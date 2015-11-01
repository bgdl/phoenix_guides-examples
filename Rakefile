require 'guidedown'

task :generate do
  %w(channels).each do |guide|
    Dir.chdir guide

    hashes = `git log --pretty=format:%H --reverse`.split

    output = ''
    initial_branch = `git branch`.split.last

    hashes.each do |hash|
      `git checkout #{hash}`

      output << Guidedown.new(
        `git show --format=%b --no-patch`,
        no_filenames: true, sticky_info_strings: true
      ).to_s

      `git clean -df`
    end

    `git checkout #{initial_branch}`

    File.open("../#{guide}.md", 'w') do |file|
      file.write output.gsub(/\s+\z/, "\n")
    end
  end
end
