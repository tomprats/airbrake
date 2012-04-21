require 'rake'
require 'rake/testtask'
require 'rubygems/package_task'
begin
  require 'cucumber/rake/task'
rescue LoadError
  $stderr.puts "Please install cucumber: `gem install cucumber`"
  exit 1
end

desc 'Default: run unit tests.'
task :default => [:test, "cucumber:rails:all"]

desc "Clean out the tmp directory"
task :clean do
  exec "rm -rf tmp"
end

desc 'Test the airbrake gem.'
Rake::TestTask.new(:test) do |t|
  t.libs << 'lib'
  t.pattern = 'test/**/*_test.rb'
  t.verbose = true
end

namespace :changeling do
  desc "Bumps the version by a minor or patch version, depending on what was passed in."
  task :bump, :part do |t, args|
    # Thanks, Jeweler!
    if Airbrake::VERSION  =~ /^(\d+)\.(\d+)\.(\d+)(?:\.(.*?))?$/
      major = $1.to_i
      minor = $2.to_i
      patch = $3.to_i
      build = $4
    else
      abort
    end

    case args[:part]
    when /minor/
      minor += 1
      patch = 0
    when /patch/
      patch += 1
    else
      abort
    end

    version = [major, minor, patch, build].compact.join('.')

    File.open(File.join("lib", "airbrake", "version.rb"), "w") do |f|
      f.write <<EOF
module Airbrake
  VERSION = "#{version}".freeze
end
EOF
    end
  end

  desc "Writes out the new CHANGELOG and prepares the release"
  task :change do
    load 'lib/airbrake/version.rb'
    file    = "CHANGELOG"
    old     = File.read(file)
    version = Airbrake::VERSION
    message = "Bumping to version #{version}"

    File.open(file, "w") do |f|
      f.write <<EOF
Version #{version} - #{Time.now}
===============================================================================

#{`git log $(git tag | tail -1)..HEAD | git shortlog`}
#{old}
EOF
    end

    exec ["#{ENV["EDITOR"]} #{file}",
          "git commit -aqm '#{message}'",
          "git tag -a -m '#{message}' v#{version}",
          "echo '\n\n\033[32mMarked v#{version} /' `git show-ref -s refs/heads/master` 'for release. Run: rake changeling:push\033[0m\n\n'"].join(' && ')
  end

  desc "Bump by a minor version (1.2.3 => 1.3.0)"
  task :minor do |t|
    Rake::Task['changeling:bump'].invoke(t.name)
    Rake::Task['changeling:change'].invoke
  end

  desc "Bump by a patch version, (1.2.3 => 1.2.4)"
  task :patch do |t|
    Rake::Task['changeling:bump'].invoke(t.name)
    Rake::Task['changeling:change'].invoke
  end

  desc "Push the latest version and tags"
  task :push do |t|
    system("git push origin master")
    system("git push origin $(git tag | tail -1)")
  end
end

begin
  require 'yard'
  YARD::Rake::YardocTask.new do |t|
    t.files   = ['lib/**/*.rb', 'TESTING.rdoc']
  end
rescue LoadError
end

GEM_ROOT = File.dirname(__FILE__).freeze

desc "Clean files generated by rake tasks"
task :clobber => [:clobber_rdoc, :clobber_package]

LOCAL_GEM_ROOT = File.join(GEM_ROOT, 'tmp', 'local_gems').freeze
RAILS_VERSIONS = IO.read('SUPPORTED_RAILS_VERSIONS').strip.split("\n")
LOCAL_GEMS = [['sham_rack', nil], ['capistrano', nil], ['sqlite3-ruby', nil], ['sinatra', nil], ['rake', '0.8.7']] +
  RAILS_VERSIONS.collect { |version| ['rails', version] }

desc "Vendor test gems: Run this once to prepare your test environment"
task :vendor_test_gems do
  old_gem_path = ENV['GEM_PATH']
  old_gem_home = ENV['GEM_HOME']
  ENV['GEM_PATH'] = LOCAL_GEM_ROOT
  ENV['GEM_HOME'] = LOCAL_GEM_ROOT
  LOCAL_GEMS.each do |gem_name, version|
    gem_file_pattern = [gem_name, version || '*'].compact.join('-')
    version_option = version ? "-v #{version}" : ''
    pattern = File.join(LOCAL_GEM_ROOT, 'gems', "#{gem_file_pattern}")
    existing = Dir.glob(pattern).first
    unless existing
      command = "gem install -i #{LOCAL_GEM_ROOT} --no-ri --no-rdoc --backtrace #{version_option} #{gem_name}"
      puts "Vendoring #{gem_file_pattern}..."
      unless system("#{command} 2>&1")
        puts "Command failed: #{command}"
        exit(1)
      end
    end
  end
  ENV['GEM_PATH'] = old_gem_path
  ENV['GEM_HOME'] = old_gem_home
end

Cucumber::Rake::Task.new(:cucumber) do |t|
  t.fork = true
  t.cucumber_opts = ['--format', (ENV['CUCUMBER_FORMAT'] || 'progress')]
end

task :cucumber => [:vendor_test_gems]

def run_rails_cucumbr_task(version, additional_cucumber_args)
  puts "Testing Rails #{version}"
  if version.empty?
    raise "No Rails version specified - make sure ENV['RAILS_VERSION'] is set, e.g. with `rake cucumber:rails:all`"
  end
  ENV['RAILS_VERSION'] = version
  cmd   = "cucumber --format #{ENV['CUCUMBER_FORMAT'] || 'progress'} #{additional_cucumber_args} features/rails.feature features/rails_with_js_notifier.feature" 
  puts "Running command: #{cmd}"
  system(cmd)
end

def define_rails_cucumber_tasks(additional_cucumber_args = '')
  namespace :rails do
    RAILS_VERSIONS.each do |version|
      desc "Test integration of the gem with Rails #{version}"
      task version => [:vendor_test_gems] do
        exit 1 unless run_rails_cucumbr_task(version, additional_cucumber_args)
      end
    end

    desc "Test integration of the gem with all Rails versions"
    task :all do
      results = RAILS_VERSIONS.map do |version|
        run_rails_cucumbr_task(version, additional_cucumber_args)
      end

      exit 1 unless results.all?
    end
  end
end

namespace :cucumber do
  namespace :wip do
    define_rails_cucumber_tasks('--tags @wip')
  end

  define_rails_cucumber_tasks
end

