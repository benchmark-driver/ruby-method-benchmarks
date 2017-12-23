# coding: utf-8
TEMPLATE_FILE = ENV['template'] || "benchmark-ips.erb"
RESULT_DIR = "result_ruby#{RUBY_VERSION}_r#{RUBY_REVISION}"
RETRY_SAVE_PATH = "#{RESULT_DIR}/retry.json"
RETRY_MAX = 5
RETRY_ERROR_THRESHOLD = 8.0 # Threshold to retry benchmark If the result has orver ±8.0 % error.

task :default => :generate

def cmd(command)
  $stderr.puts "\e[1;32m" + command + "\e[0m"
  system command
end

desc "Generate benchmark scripts"
task :generate do
  require 'yaml'
  require 'erb'

  template = ERB.new(File.read("template/#{TEMPLATE_FILE}"))

  Dir.glob("bench/**/*.yml") do |path|
    result_dir = File.join(RESULT_DIR, File.dirname(path).sub("bench", ""))
    @json_path = File.join(result_dir, "#{File.basename(path, ".*")}.json")

    benchmark_items = YAML.load(File.read(path))
    script = template.result(binding)

    script_dir = File.join("script", File.dirname(path).sub("bench", ""))
    mkdir_p script_dir unless File.exist?(script_dir)
    script_path = File.join(script_dir, "#{File.basename(path, ".*")}.rb")
    File.open(script_path, "w") { |io| io.puts script }
  end
end

def retry_count(path)
  retry_count = {}
  if File.exist?(RETRY_SAVE_PATH)
    retry_count = JSON.parse(File.read(RETRY_SAVE_PATH))
  end

  count = (retry_count[path] || 0) + 1
  retry_count[path] = count
  File.open(RETRY_SAVE_PATH, 'w') { |io|
    io.puts JSON.dump(retry_count)
  }

  count
end

desc "Run benchmarks"
task :benchmarks do
  require 'json'
  rm_rf RETRY_SAVE_PATH

  Dir.glob("script/**/*.rb") do |path|
    result_dir = File.join(RESULT_DIR, File.dirname(path).sub("script", ""))
    mkdir_p result_dir unless File.exist?(result_dir)

    cmd "ruby #{path}"

    json_path = File.expand_path(File.join(result_dir, "#{File.basename(path, ".*")}.json"))
    json = JSON.parse(File.read(json_path))

    do_retry = false
    count = 0
    json.each do |item|
      value = item["central_tendency"]
      error = item["error"]
      if error/value * 100 >= RETRY_ERROR_THRESHOLD
        do_retry = true
        count = retry_count(path)
        break
      end
    end

    redo if do_retry && count < RETRY_MAX
  end
end

desc "Clean"
task :clean do
  rm_rf "script"
  rm_rf RESULT_DIR
end

def generate_csv
  require 'json'
  require 'csv'
  CSV.open("#{RESULT_DIR}/summary.csv", "wb") do |csv|
    csv << ["File path", "Description", "Iterations"]
    Dir.glob("#{RESULT_DIR}/**/*.json").sort.each do |path|
      next if path == RETRY_SAVE_PATH
      json = JSON.parse(File.read(path))
      json.each do |item|
        value = item["central_tendency"]
        csv << ["\"#{path}\"", "\"#{item["name"]}\"", value]
      end
    end
  end
end

task :csv do
  generate_csv()
end
