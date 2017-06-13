require 'yaml'
require 'json'
require 'http'

Version = Struct.new(:name, :version, :name_and_version)

def counts(array)
  array.each_with_object(Hash.new(0)) { |el, counts| counts[el] += 1 }
end

task :apps_and_gems do
  output = { nodes: [], links: [] }
  Dir.glob("cache/*").each do |filename|
    file = File.read(filename)
    lockfile = Bundler::LockfileParser.new(file)
    appname = filename.gsub('cache/', '')

    output[:nodes] << { id: appname, group: 'applications' }

    lockfile.dependencies.map do |name, _|
      output[:nodes] << { id: name, group: 'gems' }
      output[:links] << { source: appname, target: name }
    end
  end

  output[:nodes].uniq!
  output[:links].uniq!

  File.write("public/network.json", JSON.pretty_generate(output))
end

task :analyse do
  direct_dependencies = []
  Dir.glob("cache/*").each do |filename|
    file = File.read(filename)
    lockfile = Bundler::LockfileParser.new(file)

    lockfile.dependencies.map do |_, d|
      spec = lockfile.specs.find { |s| s.name == d.name }
      direct_dependencies << Version.new(spec.name, spec.version, spec.to_s)
    end
  end

  stats = {
    unique_gems: direct_dependencies.uniq(&:name).size,
    unique_versions_of_gems: direct_dependencies.uniq(&:name_and_version).size,
  }

  puts stats.inspect

  output = []

  direct_dependencies.uniq(&:name).sort_by(&:name).each do |app|
    versions = counts(direct_dependencies.select { |v| v.name == app.name }.map(&:version).sort.map(&:to_s))
    output << { app_name: app.name, versions: versions}
  end

  File.write("public/versions.json", JSON.pretty_generate(output))
end

task :download do
  sh "rm cache/*"
  applications = YAML.load(HTTP.get('https://raw.githubusercontent.com/alphagov/govuk-developer-docs/master/data/applications.yml'))
  repos = applications.each do |application|
    next if application["retired"]

    repo_name = application.fetch('github_repo_name')
    url = "https://raw.githubusercontent.com/alphagov/#{repo_name}/master/Gemfile.lock"
    response = HTTP.get(url)

    if response.code == 200
      File.write("cache/#{repo_name}", response)
    else
      puts "Skipping #{repo_name}"
    end
  end
end
