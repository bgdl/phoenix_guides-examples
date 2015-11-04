require 'guidedown'

task :generate do
  %w(channels templates).each do |guide|
    path = File.expand_path(File.join(File.dirname(__FILE__), guide))
    Dir.chdir(path)

    File.open("../output/#{guide}.md", 'w') do |file|
      file.write Guidedown.new(
        File.read("../#{guide}.md"),
        no_filenames: true,
        sticky_info_strings: true
      )
    end
  end
end
