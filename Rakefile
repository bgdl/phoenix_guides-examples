require 'guidedown'

task :generate do
  %w(templates).each do |guide|
    Dir.chdir(guide)

    File.open("../output/#{guide}2.md", 'w') do |file|
      file.write Guidedown.new(
        File.read("../#{guide}.md"),
        no_filenames: true,
        sticky_info_strings: true
      )
    end
  end
end
