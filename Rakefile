require "rubygems"
require "bundler/gem_tasks"
require "rake"
require "rdoc"
require "date"
require "yaml"

#############################################################################
#
# Helper functions
#
#############################################################################

def name
  @name ||= Dir["*.gemspec"].first.split(".").first
end

def version
  JekyllImport::VERSION
end

def normalize_bullets(markdown)
  markdown.gsub(%r!\s{2}\*{1}!, "-")
end

def linkify_prs(markdown)
  markdown.gsub(%r!#(\d+)!) do |word|
    "[#{word}]({{ site.repository }}/issues/#{word.delete("#")})"
  end
end

def linkify_users(markdown)
  markdown.gsub(%r!(@\w+)!) do |username|
    "[#{username}](https://github.com/#{username.delete("@")})"
  end
end

def linkify(markdown)
  linkify_users(linkify_prs(markdown))
end

def liquid_escape(markdown)
  markdown.gsub(%r!(`{[{%].+[}%]}`)!, "{% raw %}\\1{% endraw %}")
end

def remove_head_from_history(markdown)
  index = markdown =~ %r!^##\s+\d+\.\d+\.\d+!
  markdown[index..-1]
end

def converted_history(markdown)
  remove_head_from_history(liquid_escape(linkify(normalize_bullets(markdown))))
end

#############################################################################
#
# Standard tasks
#
#############################################################################

task :default => :test

require "rake/testtask"
Rake::TestTask.new(:test) do |test|
  test.libs << "lib" << "test"
  test.pattern = "test/**/test_*.rb"
  test.verbose = true
  test.warning = false
end

require "rdoc/task"
Rake::RDocTask.new do |rdoc|
  rdoc.rdoc_dir = "rdoc"
  rdoc.title = "#{name} #{version}"
  rdoc.rdoc_files.include("README*")
  rdoc.rdoc_files.include("lib/**/*.rb")
end

desc "Open an irb session preloaded with this library"
task :console do
  sh "irb -rubygems -r ./lib/#{name}.rb"
end

#############################################################################
#
# Site tasks - http://import.jekyllrb.com
#
#############################################################################

namespace :site do
  desc "Generate and view the site locally"
  task :preview => :generate_dependency_data do
    require "launchy"

    # Yep, it's a hack! Wait a few seconds for the Jekyll site to generate and
    # then open it in a browser. Someday we can do better than this, I hope.
    Thread.new do
      sleep 4
      puts "Opening in browser..."
      Launchy.open("http://localhost:4000")
    end

    # Generate the site in server mode.
    puts "Running Jekyll..."
    Dir.chdir("docs") do
      sh "jekyll serve --watch"
    end
  end

  desc "Update normalize.css library to the latest version and minify"
  task :update_normalize_css do
    normalize_file = "docs/_sass/normalize.scss"
    sh "curl \"http://necolas.github.io/normalize.css/latest/normalize.css\" -o #{normalize_file}"
    sh "sass #{normalize_file}:#{normalize_file} --style compressed"
  end

  desc "Commit the local site to the gh-pages branch and publish to GitHub Pages"
  task :publish => [:history] do
    puts "We now publish directly from the docs/ subfolder on the master branch."
  end

  desc "Create a nicely formatted history page for the jekyll site based on the repo history."
  task :history do
    if File.exist?("History.markdown")
      history_file = File.read("History.markdown")
      front_matter = {
        "layout"       => "docs",
        "title"        => "History",
        "permalink"    => "/docs/history/",
        "prev_section" => "contributing",
      }
      Dir.chdir("docs/_docs/") do
        File.open("history.md", "w") do |file|
          file.write("#{front_matter.to_yaml}---\n\n")
          file.write(converted_history(history_file))
        end
      end
      if `git diff docs/_docs/history.md`.strip.empty?
        puts "No updates to commit at this time. Skipping..."
      else
        sh "git add docs/_docs/history.md"
        sh "git commit -m 'Updated generated history.md file in the site.'"
      end
    else
      abort "You seem to have misplaced your History.markdown file. I can haz?"
    end
  end

  desc "generate importer-dependencies data file"
  task :generate_dependency_data do
    require "jekyll-import"

    # Monkey-patch to alter behavior just for this task.
    module JekyllImport
      def self.require_with_fallback(gems)
        Array(gems).flatten
      end
    end

    data_dir   = File.expand_path("docs/_data", __dir__)
    data_file  = File.join(data_dir, "importer_dependencies.yml")
    importers  = JekyllImport::Importer.subclasses
    label_size = importers.map { |klass| klass.to_s.split("::").last.length }.max + 1

    # available as part of Ruby Stdlib
    std_lib = %w(csv date fileutils json net/http open-uri rss rubygems time uri yaml)

    FileUtils.mkdir_p data_dir
    data = {}

    puts "\nLogging importer dependencies..\n\n"
    importers.each do |importer|
      doc_name = importer.to_s.split("::").last.downcase
      deps = importer.require_deps - std_lib
      next if deps.empty?

      deps.map! { |dep| dep.start_with?("active_support") ? "activesupport" : dep.split("/")[0] }
      deps = deps - std_lib
      deps.uniq!
      deps.sort!
      puts "#{doc_name.ljust(label_size)}: #{deps.join(", ")}"
      data[doc_name] = deps
    end

    File.write(data_file, YAML.dump(data))
    puts <<~MSG
      \nData successfully logged into file:
        => #{data_file}
      Commit the file into version control if altered.
    MSG
  end
end
