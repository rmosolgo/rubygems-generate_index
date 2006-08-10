# Rakefile for RubyGems      -*- ruby -*-

#--
# Copyright 2006 by Chad Fowler, Rich Kilmer, Jim Weirich and others.
# All rights reserved.
# See LICENSE.txt for permissions.
#++

require 'rubygems'
require 'rake/clean'
require 'rake/testtask'
require 'rake/packagetask'
require 'rake/gempackagetask'
require 'rake/rdoctask'

def announce(msg='')
  STDERR.puts msg
end

# Disable certs for now. 
ENV.delete('CERT_DIR')

PKG_NAME = 'rubygems'
def package_version
  `ruby -Ilib bin/gem environment packageversion`.chomp
end

if ENV['REL']
  PKG_VERSION = ENV['REL']
  CURRENT_VERSION = package_version
else
  PKG_VERSION = package_version
  CURRENT_VERSION = PKG_VERSION
end

CLEAN.include("COMMENTS")
CLOBBER.include(
  "test/data/one/one-*0.0.1.gem",
  "test/temp",
  'test/mock/gem/doc',
  'test/mock/gem/cache',
  'test/data/gemhome',
  'test/data/[a-z]*.gem',
  'scripts/*.hieraki',
  'data__',
  'html',
  'pkgs/sources/sources*.gem',
  '.config',
  '**/debug.log',
  'logs'
  )

task :default => [:test]
task :test => [:test_units]

Rake::TestTask.new(:test_units) do |t|
  t.test_files = FileList['test/test*.rb']
end

Rake::TestTask.new(:test_functional) do |t|
  t.test_files = FileList['test/functional*.rb']
end

Rake::TestTask.new(:test_all) do |t|
  t.test_files = FileList['test/{test,functional}*.rb']
end

desc "Run the tests for a build"
task :build_tests do
  html_dir = ENV['TESTRESULTS'] || 'html/tests'
  ruby %{-Ilib scripts/buildtests.rb #{html_dir}}
  open("#{html_dir}/summary.html") do |inf|
    open("#{html_dir}/summary.new", "w") do |outf|
      inf.each do |line|
	if line =~ /td align/
	  line = "    <td align=\"left\">#{Time.now}</td><td align=\"right\">"
	end
	outf.puts line
      end
    end
  end
  mv "#{html_dir}/summary.html", "#{html_dir}/summary.old"
  mv "#{html_dir}/summary.new", "#{html_dir}/summary.html"      
end

# Shortcuts for test targets
task :tf => [:test_functional]
task :tu => [:test_units]
task :ta => [:test_all]

task :gemtest do
  ruby %{-Ilib -rscripts/runtest -e 'run_tests("test/test_gempaths.rb", true)'}
end

# --------------------------------------------------------------------
# Creating a release

desc "Make a new release"
task :release => [
  :prerelease,
  :clobber,
  :test_all,
  :update_version,
  :package,
  :tag] do
  
  announce 
  announce "**************************************************************"
  announce "* Release #{PKG_VERSION} Complete."
  announce "* Packages ready to upload."
  announce "**************************************************************"
  announce 
end

# Validate that everything is ready to go for a release.
task :prerelease do
  announce 
  announce "**************************************************************"
  announce "* Making RubyGem Release #{PKG_VERSION}"
  announce "* (current version #{CURRENT_VERSION})"
  announce "**************************************************************"
  announce  

  # Is a release number supplied?
  unless ENV['REL']
    fail "Usage: rake release REL=x.y.z [REUSE=tag_suffix]"
  end

  # Is the release different than the current release.
  # (or is REUSE set?)
  if PKG_VERSION == CURRENT_VERSION && ! ENV['REUSE']
    fail "Current version is #{PKG_VERSION}, must specify REUSE=tag_suffix to reuse version"
  end

  # Are all source files checked in?
  if ENV['RELTEST']
    announce "Release Task Testing, skipping checked-in file test"
  else
    announce "Checking for unchecked-in files..."
    data = `cvs -q update`
    unless data =~ /^$/
      fail "CVS update is not clean ... do you have unchecked-in files?"
    end
    announce "No outstanding checkins found ... OK"
  end
end

task :update_version => [:prerelease] do
  if PKG_VERSION == CURRENT_VERSION
    announce "No version change ... skipping version update"
  else
    announce "Updating RubyGem version to #{PKG_VERSION}"
    open("lib/rubygems/rubygems_version.rb", "w") do |f|
      f.puts "# DO NOT EDIT"
      f.puts "# This file is auto-generated by build scripts."
      f.puts "# See:  rake update_version"
      f.puts "module Gem"
      f.puts "  RubyGemsVersion = '#{PKG_VERSION}'"
      f.puts "end"
    end
    if ENV['RELTEST']
      announce "Release Task Testing, skipping commiting of new version"
    else
      sh %{cvs commit -m "Updated to version #{PKG_VERSION}" lib/rubygems/rubygems_version.rb} # "
    end
  end
end

task :tag => [:prerelease] do
  reltag = "REL_#{PKG_VERSION.gsub(/\./, '_')}"
  reltag << ENV['REUSE'].gsub(/\./, '_') if ENV['REUSE']
  announce "Tagging CVS with [#{reltag}]"
  if ENV['RELTEST']
    announce "Release Task Testing, skipping CVS tagging"
  else
    sh %{cvs tag #{reltag}}
  end
end

# --------------------------------------------------------------------

begin
  require 'rcov/rcovtask'
  HAVE_RCOV = true
rescue LoadError
  HAVE_RCOV = false
end

if HAVE_RCOV
  Rcov::RcovTask.new do |t|
    t.libs << "test"
    t.rcov_opts = ['-xRakefile', '-xrakefile', '-xpublish.rf', '--text-report']
    t.test_files = FileList[
      'test/test*.rb'
    ]
    t.verbose = true
  end
end

# --------------------------------------------------------------------
# Create a task to build the RDOC documentation tree.

desc "Create the RDOC html files"
rd = Rake::RDocTask.new("rdoc") { |rdoc|
  rdoc.rdoc_dir = 'html'
  rdoc.title    = "RubyGems"
  rdoc.options << '--line-numbers' << '--inline-source' << '--main' << 'README'
  rdoc.rdoc_files.include('README', 'TODO', 'Releases', 'LICENSE.txt', 'GPL.txt')
  rdoc.rdoc_files.include('lib/**/*.rb')
#  rdoc.rdoc_files.include('test/**/*.rb')
}

desc "Publish the RDOCs on RubyForge"
task :publish_rdoc => ["html/index.html"] do
  # NOTE: This task assumes that you have an SSH alias setup for rubyforge.
  mkdir_p "emptydir"
  sh "scp -rq emptydir rubyforge:/var/www/gforge-projects/rubygems/rdoc"
  sh "scp -rq html/* rubyforge:/var/www/gforge-projects/rubygems/rdoc"
  rm_r "emptydir"
end

# Wiki Doc Targets

desc "Upload the Hieraki Data"
task :upload => [:upload_gemdoc]

task :upload_gemdoc => ['scripts/gemdoc.hieraki'] do
  ruby %{scripts/upload_gemdoc.rb}
end

desc "Build the Hieraki documentation"
task :hieraki => ['scripts/gemdoc.hieraki', 'scripts/specdoc.hieraki']

file 'scripts/gemdoc.hieraki' => ['scripts/gemdoc.rb', 'scripts/gemdoc.data'] do
  chdir('scripts') do
    ruby %{-I../lib gemdoc.rb <gemdoc.data >gemdoc.hieraki}
  end
end

file 'scripts/specdoc.hieraki' =>
  ['scripts/specdoc.rb', 'scripts/specdoc.data', 'scripts/specdoc.yaml'] do
  chdir('scripts') do
    ruby %{-I../lib specdoc.rb >specdoc.hieraki}
  end
end

# Package tasks

PKG_FILES = FileList[
  "Rakefile", "ChangeLog", "Releases", "TODO", "README", 
  "setup.rb",
  "post-install.rb",
  "bin/*",
  "doc/*.css", "doc/*.rb",
  "examples/**/*",
  "gemspecs/**/*",
  "lib/**/*.rb",
  "pkgs/**/*",
  "redist/*.gem",
  "scripts/*.rb",
  "test/**/*"
]
PKG_FILES.exclude(%r(^test/temp(/|$)))

Rake::PackageTask.new("package") do |p|
  p.name = PKG_NAME
  p.version = PKG_VERSION
  p.need_tar = true
  p.need_zip = true
  p.package_files = PKG_FILES
end

Spec = Gem::Specification.new do |s|
  s.name = PKG_NAME + "-update"  
  s.version = PKG_VERSION
  s.summary = "RubyGems Update GEM"
  s.description = %{RubyGems is a package management framework for Ruby.  This Gem
is a update for the base RubyGems software.  You must have a base
installation of RubyGems before this update can be applied.
}
  s.files = PKG_FILES.to_a
  s.require_path = 'lib'
  s.authors = ['Jim Weirich', 'Chad Fowler']
  s.email = "rubygems-developers@rubyforge.org"
  s.homepage = "http://rubygems.rubyforge.org"
  s.rubyforge_project = "rubygems"
  s.bindir = "bin"                               # Use these for applications.
  s.executables = ["update_rubygems"]
  certdir = ENV['CERT_DIR']
  if certdir
    s.signing_key = File.join(certdir, 'gem-private_key.pem')
    s.cert_chain  = [File.join(certdir, 'gem-public_cert.pem')]
  end
end

# Add console output about signing the Gem
file "pkg/#{Spec.full_name}.gem" do
  puts "Signed with certificates in '#{ENV['CERT_DIR']}'" if ENV['CERT_DIR']
end

Rake::GemPackageTask.new(Spec) do |p| end

desc "Build the Gem spec file for the rubygems-update package"
task :gemspec => "pkg/rubygems-update.gemspec"
file "pkg/rubygems-update.gemspec" => ["pkg", "Rakefile"] do |t|
  open(t.name, "w") do |f| f.puts Spec.to_yaml end
end

# Install RubyGems

desc "Install RubyGems"
task :install do
  ruby 'setup.rb config'
  ruby 'setup.rb setup'
  ruby 'setup.rb install'
end

# Run 'gem' (using local bin and lib directories).
# e.g.
#     rake rungem -- install -r blahblah --test

desc "Run local 'gem'"
task :rungem do
  ARGV.shift
  exec "ruby -Ilib bin/gem #{ARGV.join(' ')}"
end

# Misc Tasks ---------------------------------------------------------

def egrep(pattern)
  Dir['**/*.rb'].each do |fn|
    count = 0
    open(fn) do |f|
      while line = f.gets
	count += 1
	if line =~ pattern
	  puts "#{fn}:#{count}:#{line}"
	end
      end
    end
  end
end

desc "Look for TODO and FIXME tags in the code"
task :todo do
  egrep /#.*(FIXME|TODO|TBD)/
end

desc "Look for Debugging print lines"
task :dbg do
  egrep /\bDBG|\bbreakpoint\b/
end

desc "List all ruby files"
task :rubyfiles do 
  puts Dir['**/*.rb'].reject { |fn| fn =~ /^pkg/ }
  puts Dir['bin/*'].reject { |fn| fn =~ /CVS|(~$)|(\.rb$)/ }
end

task :rf => :rubyfiles
