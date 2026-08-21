desc "Fetch GitHub projects and write to _data/projects.yml"
task :projects do
  require 'net/http'
  require 'json'
  require 'yaml'
  require 'time'

  username = 'wassim31'

  # Repos not relevant to the projects page
  exclude = %w[
    wassim31
    wassim31.github.io
  ]

  repos = []
  page = 1

  loop do
    uri = URI("https://api.github.com/users/#{username}/repos?per_page=100&page=#{page}&sort=pushed")
    request = Net::HTTP::Get.new(uri)
    request['Accept'] = 'application/vnd.github.v3+json'
    request['User-Agent'] = 'wassim31.github.io-project-fetcher'
    request['Authorization'] = "token #{ENV['GITHUB_TOKEN']}" if ENV['GITHUB_TOKEN']

    response = Net::HTTP.start(uri.hostname, uri.port, use_ssl: true) { |http| http.request(request) }
    batch = JSON.parse(response.body)
    break if batch.empty? || batch.is_a?(Hash)
    repos.concat(batch)
    break if batch.length < 100
    page += 1
  end

  filtered = repos.select do |repo|
    next false if repo['fork']
    next false if repo['archived']
    next false if repo['description'].nil? || repo['description'].empty?
    next false if exclude.include?(repo['name'])
    true
  end

  filtered.sort_by! { |r| -r['stargazers_count'] }

  projects = filtered.map do |repo|
    {
      'name' => repo['name'],
      'url' => repo['html_url'],
      'description' => repo['description'],
      'stars' => repo['stargazers_count'],
      'language' => repo['language'],
      'pushed_at' => repo['pushed_at']
    }
  end

  File.write('_data/projects.yml', projects.to_yaml)
  puts "Wrote #{projects.length} projects to _data/projects.yml"
end
