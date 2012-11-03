PROJECT_VERSION = '1.28'
PLATFORMS_DIR = (ENV['PLATFORMS_DIR'] || '/Applications/Xcode.app/Contents/Developer/Platforms')

sim_sdks = Dir.glob(File.join(PLATFORMS_DIR, 'iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator*.sdk')).map do |path|
  File.basename(path).scan(/^iPhoneSimulator(.+)\.sdk$/)[0][0]
end
ios_sdks = Dir.glob(File.join(PLATFORMS_DIR, 'iPhoneOS.platform/Developer/SDKs/iPhoneOS*.sdk')).map do |path|
  File.basename(path).scan(/^iPhoneOS(.+)\.sdk$/)[0][0]
end
SDK_VERSIONS = (sim_sdks & ios_sdks)

if SDK_VERSIONS.empty?
  $stderr.puts "Can't locate any SDK"
  exit 1
end

verbose(true)

def rake(dir, cmd='all')
  Dir.chdir(dir) do
    debug = ENV['DEBUG'] ? 'optz_level=0' : ''
    trace = Rake.application.options.trace
    sh "rake platforms_dir=\"#{PLATFORMS_DIR}\" sdk_versions=\"#{SDK_VERSIONS.join(',')}\" project_version=\"#{PROJECT_VERSION}\" #{debug} #{cmd} #{trace ? '--trace' : ''}"
  end
end

targets = %w{vm bin lib data doc}

task :default => :all
desc "Build everything"
task :all => :build

targets.each do |target|
  desc "Build target #{target}"
  task "build:#{target}" do
    rake(target)
  end
end

desc "Build all targets"
task :build => targets.map { |x| "build:#{x}" }

targets.each do |target|
  desc "Clean target #{target}"
  task "clean:#{target}" do
    rake(target, 'clean')
  end
end

desc "Clean all targets"
task :clean => targets.map { |x| "clean:#{x}" }

desc "Generate source code archive"
task :archive do
  base = "rubymotion-head"
  rm_rf "/tmp/#{base}"
  sh "git archive --format=tar --prefix=#{base}/ HEAD | (cd /tmp && tar xf -)"
  Dir.chdir('vm') do
    sh "git archive --format=tar HEAD | (cd /tmp/#{base}/vm && tar xf -)"
  end
  Dir.chdir('/tmp') do
    sh "tar -czf #{base}.tgz #{base}"
  end
  sh "mv /tmp/#{base}.tgz ."
  sh "du -h #{base}.tgz"
end

desc "Install"
task :install do
  public_binaries = ['./bin/motion']
  binaries = public_binaries.dup.concat(['./bin/deploy', './bin/sim',
    './bin/llc', './bin/ruby', './bin/ctags'])
  data = ['./NEWS']
  data.concat(Dir.glob('./lib/motion/**/*'))
  SDK_VERSIONS.each do |sdk_version|
    data.concat(Dir.glob("./data/#{sdk_version}/BridgeSupport/*.bridgesupport"))
    data.concat(Dir.glob("./data/#{sdk_version}/iPhoneOS/*"))
    data.concat(Dir.glob("./data/#{sdk_version}/iPhoneSimulator/*"))
  end

  # === 6.0 support (beta) ===
  data.concat(Dir.glob("./data/6.0/Rakefile"))
  data.concat(Dir.glob("./data/6.0/BridgeSupport/RubyMotion.bridgesupport"))
  data.concat(Dir.glob("./data/6.0/BridgeSupport/UIAutomation.bridgesupport"))
  data.concat(Dir.glob("./data/6.0/iPhoneOS/*"))
  data.concat(Dir.glob("./data/6.0/iPhoneSimulator/*"))
  # ==========================

  data.concat(Dir.glob('./data/*-ctags.cfg'))
  #data.concat(Dir.glob('./doc/*.html'))
  #data.concat(Dir.glob('./doc/docset/**/*'))
  #data.concat(Dir.glob('./sample/**/*').reject { |path| path =~ /build/ })
  data.reject! { |path| /^\./.match(File.basename(path)) }

  motiondir = '/Library/RubyMotion'
  destdir = (ENV['DESTDIR'] || '/')
  destmotiondir = File.join(destdir, motiondir)
  install = proc do |path, mode|
    pathdir = File.join(destmotiondir, File.dirname(path))
    mkdir_p pathdir unless File.exist?(pathdir)
    destpath = File.join(destmotiondir, path)
    if File.directory?(path)
      mkdir_p destpath
    else
      cp path, destpath
      chmod mode, destpath
    end
    destpath
  end

  binaries.each { |path| install.call(path, 0755) }
  data.each { |path| install.call(path, 0644) }

  bindir = File.join(destdir, '/usr/bin')
  mkdir_p bindir
  public_binaries.each do |path|
    destpath = File.join(motiondir, path)
    ln_sf destpath, File.join(bindir, File.basename(path))
  end

=begin
  # Gems (only for beta).
  gemsdir = File.join(destmotiondir, 'gems')
  mkdir_p gemsdir
  cp '../motion-testflight/pkg/motion-testflight-1.0.gem', gemsdir
=end
end

desc "Generate .pkg"
task :package do
  destdir = '/tmp/Motion'
  pkg = "pkg/RubyMotion #{PROJECT_VERSION}.pkg"
  #if !File.exist?(destdir) or !File.exist?(pkg) or File.mtime(destdir) > File.mtime(pkg)
    ENV['DESTDIR'] = destdir
    rm_rf destdir
    Rake::Task[:install].invoke

    sh "/Applications/PackageMaker.app/Contents/MacOS/PackageMaker --doc pkg/RubyMotion.pmdoc --out \"pkg/RubyMotion #{PROJECT_VERSION}.pkg\" --version #{PROJECT_VERSION}"
  #end
end

desc "Push on Amazon S3"
task :upload do
  require 'rubygems'
  require 'aws/s3'
  require 'yaml'

  s3config = YAML.load(File.read('s3config.yaml'))

  AWS::S3::Base.establish_connection!(
    :access_key_id => s3config[:access_key_id],
    :secret_access_key => s3config[:secret_access_key]
  )

  WEBSITE_BUCKET_NAME = 'data.hipbyte.com'

  # Will raise an error if bucket doesn't exist
  AWS::S3::Bucket.find WEBSITE_BUCKET_NAME

  file = "pkg/RubyMotion #{PROJECT_VERSION}.pkg"
  puts "Uploading #{file}.."
  AWS::S3::S3Object.store("rubymotion/releases/#{PROJECT_VERSION}.pkg", File.read(file), WEBSITE_BUCKET_NAME)
  puts "Done!"

  puts "Uploading Latest.."
  AWS::S3::S3Object.store('rubymotion/releases/Latest', PROJECT_VERSION, WEBSITE_BUCKET_NAME)
  puts "Done!"
end

namespace :doc do
  require './doc/docset'
  require 'fileutils'

  YARDOC = "cd vm; bundle exec yardoc --legacy"
  RUBY_SOURCES = %w{
    array.c bignum.c class.c compar.c complex.c dir.c encoding.c enum.c
    enumerator.c env.c error.c eval.c eval_error.c eval_jump.c eval_safe.c
    file.c hash.c io.c kernel.c load.c marshal.c math.c numeric.c object.c
    pack.c prec.c proc.c process.c random.c range.c rational.c re.c
    signal.c sprintf.c string.c struct.c symbol.c thread.c time.c
    transcode.c ucnv.c util.c variable.c vm.cpp vm_eval.c vm_method.c
    NSArray.m NSDictionary.m NSString.m bridgesupport.cpp gcd.c objc.m sandbox.c
  }
  DOCSET = {
    "UIKit" => "~/Library/Developer/Shared/Documentation/DocSets/com.apple.adc.documentation.AppleiOS6.0.iOSLibrary.docset/Contents/Resources/Documents/documentation/UIKit/Reference",
    "Foundation" => "~/Library/Developer/Shared/Documentation/DocSets/com.apple.adc.documentation.AppleiOS6.0.iOSLibrary.docset/Contents/Resources/Documents/documentation/Cocoa/Reference"
  }
  OUTPUT_DIR = "api"
  DOCSET_RUBY_FILES_DIR = '/tmp/rb_docset'

  desc "Generate API Documents"
  task :api => [:"list:framework"] do
    rubymotion_files = RUBY_SOURCES.join(" ")

    # generate Ruby code from iOS SDK docset
    docset_paths = []
    DOCSET.each_value{ |x| docset_paths << x }
    DocsetGenerator.new('docset', docset_paths).generate_ruby_code

    docset_files = Dir.glob(File.join(DOCSET_RUBY_FILES_DIR, '*.rb')).join(" ")
    frameworks   = Dir.glob(File.join(OUTPUT_DIR, '*.md')).map{ |x| "../#{x}" }.join(" ")

    sh "#{YARDOC} --title 'RubyMotion API Reference' -o ../#{OUTPUT_DIR} #{rubymotion_files} #{docset_files} - ../doc/RubyMotion.md #{frameworks}"
    FileUtils.ln "#{OUTPUT_DIR}/_index.html", "#{OUTPUT_DIR}/index.html"
  end

  namespace :list do
    def save_class_list(title, doc_output_dir, output_file_path)
      doc_files = Dir.glob(File.join(doc_output_dir, '**/*.html')).sort.map { |x| x.sub("#{doc_output_dir}/", '')}
      File.open(output_file_path, "w") { |io|
        io.puts "# @markup markdown"
        io.puts "# @title #{title}"
        io.puts "# #{title}"

        doc_files.each do |file|
          next if file =~ /index.html/
          next if file =~ /_list.html/
          next if file == "frames.html"
          next if file == "top-level-namespace.html"

          class_name = file.sub(/(\w+)\/(\w+)/, '\1::\2')
          class_name = class_name.sub('.html', '')
          io.puts "- [#{class_name}](#{file})"
        end
      }
    end

    def save_framework_class_list(name, path)
      # generate Ruby code from iOS SDK docset
      FileUtils.rm_rf OUTPUT_FRAMEWORK_DOC_DIR
      FileUtils.rm_rf DOCSET_RUBY_FILES_DIR
      DocsetGenerator.new('docset', [path]).generate_ruby_code

      docset_files = Dir.glob(File.join(DOCSET_RUBY_FILES_DIR, '*.rb')).join(" ")
      sh "#{YARDOC} -o #{OUTPUT_FRAMEWORK_DOC_DIR} #{docset_files}"

      save_class_list("#{name}", OUTPUT_FRAMEWORK_DOC_DIR, "#{OUTPUT_DIR}/#{name}.md")
    end

    desc "Generate RubyMotion Classes List"
    task :rubymotion do
      OUTPUT_RUBY_DOC_DIR = '/tmp/rubymotion_doc'

      rubymotion_files = RUBY_SOURCES.join(" ")

      FileUtils.rm_rf OUTPUT_RUBY_DOC_DIR
      sh "#{YARDOC} -o #{OUTPUT_RUBY_DOC_DIR} #{rubymotion_files}"
      save_class_list("RubyMotion", OUTPUT_RUBY_DOC_DIR, "doc/RubyMotion.md")
    end

    desc "Generate iOS SDK Framework Classes List"
    task :framework do
      OUTPUT_FRAMEWORK_DOC_DIR = '/tmp/rb_framework_doc'

      FileUtils.rm_rf OUTPUT_DIR
      FileUtils.mkdir_p OUTPUT_DIR

      save_framework_class_list('UIKit', DOCSET['UIKit'])
      save_framework_class_list('Foundation', DOCSET['Foundation'])
    end
  end
end