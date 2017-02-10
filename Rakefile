require 'date'

namespace :generate do
  desc 'Generate new post'
  task :post do
    title, file_title = ask_for_titles
    STDOUT.puts "Writing file #{new_post_filename(file_title)}"
    File.write(new_post_filename(file_title), new_post_content(title))
  end
end

def ask_for_titles
  STDOUT.write 'Title: '
  title = STDIN.gets.chomp

  STDOUT.write 'Title for file (split words by dash): '
  file_title = STDIN.gets.chomp.downcase

  [title, file_title]
end

def new_post_filename(title)
  "_posts/#{Date.today}-#{title}.md"
end

def new_post_content(title)
<<-TEMPLATE
---

title:  '#{title}'
categories: development
tags: [ruby, notes]
---

## Intro

Here you go

<!--more-->

And here too.
TEMPLATE
end
