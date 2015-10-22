## Tying it all together
Let's tie all these ideas together by building a simple chat application. After [generating a new Phoenix application](http://www.phoenixframework.org/docs/up-and-running) we'll see that the endpoint is already set up for us in `lib/hello_phoenix/endpoint.ex`:

```elixir
defmodule HelloPhoenix.Endpoint do
  use Phoenix.Endpoint, otp_app: :hello_phoenix

  socket "/socket", HelloPhoenix.UserSocket
  ...
end
```

In `web/channels/user_socket.ex`, the `HelloPhoenix.UserSocket` we pointed to in our endpoint has already been created when we generated our application. We need to make sure messages get routed to the correct channel. To do that, we'll uncomment the "rooms:*" channel definition:

```elixir
defmodule HelloPhoenix.UserSocket do
  use Phoenix.Socket

  ## Channels
  channel "rooms:*", HelloPhoenix.RoomChannel
  ...
```

Now, whenever a client sends a message whose topic starts with `"rooms:"`, it will be routed to our RoomChannel. Next, we'll define a `HelloPhoenix.RoomChannel` module to manage our chat room messages.

### Joining Channels

The first priority of your channels is to authorize clients to join a given topic. For authorization, we must implement `join/3` in `web/channels/room_channel.ex`.

```elixir
defmodule HelloPhoenix.RoomChannel do
  use Phoenix.Channel

  def join("rooms:lobby", _message, socket) do
    {:ok, socket}
  end
  def join("rooms:" <> _private_room_id, _params, _socket) do
    {:error, %{reason: "unauthorized"}}
  end
end
```

For our chat app, we'll allow anyone to join the `"rooms:lobby"` topic, but any other room will be considered private and special authorization, say from a database, will be required. We won't worry about private chat rooms for this exercise, but feel free to explore after we finish. To authorize the socket to join a topic, we return `{:ok, socket}` or `{:ok, reply, socket}`. To deny access, we return `{:error, reply}`. More information about authorization with tokens can be found in the [`Phoenix.Token` documentation](http://hexdocs.pm/phoenix/Phoenix.Token.html).

With our channel in place, let's get the client and server talking. There's some code in `web/static/js/socket.js` to connect to our socket and join our channel already, we just need to set our room name to "rooms:lobby".

```javascript
...
socket.connect()

// Now that you are connected, you can join channels with a topic:
let channel = socket.channel("rooms:lobby", {})
channel.join()
  .receive("ok", resp => { console.log("Joined successfully", resp) })
  .receive("error", resp => { console.log("Unable to join", resp) })

export default socket
```

After that, we need to make sure `web/static/socket.js` gets imported into our application javascript file. To do that, uncomment the last line in `web/static/js/app.js`.

```javascript
...
import socket from "./socket"
```

Save the file and your browser should auto refresh, thanks to the Phoenix live reloader. If everything worked, we should see "Joined successfully" in the browser's JavaScript console. Our client and server are now talking over a persistent connection. Now let's make it useful by enabling chat.

In `web/templates/page/index.html.eex`, we'll replace the existing code with a container to hold our chat messages, and an input field to send them:

```
<div id="messages"></div>
<input id="chat-input" type="text"></input>
```

We'll also add jQuery to our application layout in `web/templates/layout/app.html.eex`:

```
      ...

      <%= @inner %>

    </div> <!-- /container -->
    <script src="//code.jquery.com/jquery-1.11.3.min.js"></script>
    <script src="<%= static_path(@conn, "/js/app.js") %>"></script>
  </body>
</html>
```

Now let's add a couple of event listeners to `web/static/js/socket.js`:

```javascript
...
let channel           = socket.channel("rooms:lobby", {})
let chatInput         = $("#chat-input")
let messagesContainer = $("#messages")

chatInput.on("keypress", event => {
  if(event.keyCode === 13){
    channel.push("new_msg", {body: chatInput.val()})
    chatInput.val("")
  }
})

channel.join()
  .receive("ok", resp => { console.log("Joined successfully", resp) })
  .receive("error", resp => { console.log("Unable to join", resp) })

export default socket
```

All we had to do is detect that enter was pressed and then `push` an event over the channel with the message body. We named the event "new_msg". With this in place, let's handle the other piece of a chat application where we listen for new messages and append them to our messages container.

```javascript
...
let channel           = socket.channel("rooms:lobby", {})
let chatInput         = $("#chat-input")
let messagesContainer = $("#messages")

chatInput.on("keypress", event => {
  if(event.keyCode === 13){
    channel.push("new_msg", {body: chatInput.val()})
    chatInput.val("")
  }
})

channel.on("new_msg", payload => {
  messagesContainer.append(`<br/>[${Date()}] ${payload.body}`)
})

channel.join()
  .receive("ok", resp => { console.log("Joined successfully", resp) })
  .receive("error", resp => { console.log("Unable to join", resp) })

export default socket
```

We listen for the `"new_msg"` event using `channel.on`, and then append the message body to the DOM. Now let's handle the incoming and outgoing events on the server to complete the picture.

### Incoming Events
We handle incoming events with `handle_in/3`. We can pattern match on the event names, like `"new_msg"`, and then grab the payload that the client passed over the channel. For our chat application, we simply need to notify all other `rooms:lobby` subscribers of the new message with `broadcast!/3`.

```elixir
defmodule HelloPhoenix.RoomChannel do
  use Phoenix.Channel

  def join("rooms:lobby", _message, socket) do
    {:ok, socket}
  end
  def join("rooms:" <> _private_room_id, _params, _socket) do
    {:error, %{reason: "unauthorized"}}
  end

  def handle_in("new_msg", %{"body" => body}, socket) do
    broadcast! socket, "new_msg", %{body: body}
    {:noreply, socket}
  end

  def handle_out("new_msg", payload, socket) do
    push socket, "new_msg", payload
    {:noreply, socket}
  end
end
```

`broadcast!/3` will notify all joined clients on this `socket`'s topic and invoke their `handle_out/3` callbacks. `handle_out/3` isn't a required callback, but it allows us to customize and filter broadcasts before they reach each client. By default, `handle_out/3` is implemented for us and simply pushes the message on to the client, just like our definition. We included it here because hooking into outgoing events allows for powerful message customization and filtering. Let's see how.

#### Intercepting Outgoing Events
We won't implement this for our application, but imagine our chat app allowed users to ignore messages about new users joining a room. We could implement that behavior like this where we explicitly tell Phoenix which outgoing event we want to intercept and then define a `handle_out/3` callback for those events. (Of course, this assumes that we have a `User` model with an `ignoring?/2` function, and that we pass a user in via the `assigns` map.)

```elixir
intercept ["user_joined"]

def handle_out("user_joined", msg, socket) do
  if User.ignoring?(socket.assigns[:user], msg.user_id) do
    {:noreply, socket}
  else
    push socket, "user_joined", msg
    {:noreply, socket}
  end
end
```

That's all there is to our basic chat app. Fire up multiple browser tabs and you should see your messages being pushed and broadcasted to all windows!

#### Socket Assigns

Similar to connection structs, `%Plug.Conn{}`, it is possible to assign values to a channel socket. `Phoenix.Socket.assign/3` is conveniently imported into a channel module as `assign/3`:

```elixir
socket = assign(socket, :user, msg["user"])
```

Sockets store assigned values as a map in `socket.assigns`.

#### Example Application
To see an example of the application we just built, checkout this project (https://github.com/chrismccord/phoenix_chat_example).

You can also see a live demo at (http://phoenixchat.herokuapp.com/).

 
