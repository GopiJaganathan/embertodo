require 'net/http'
require 'net/https'
require 'uri'
require 'json'
require 'yaml'
require 'pry'

module Clients
  class GitHubClient
    @url = ''
    @user = ''
    @password = ''
    @run_id = ''

    @head = ''
    @base = ''



    def initialize
      @token = "token 6f09364150f178aaf2e3dd1ce1335a2dac46bf3f"
      @owner = "GopiJaganathan"
      @repo = "embertodo"
      @testrail_url="https://api.github.com/repos/"
      @url = "#{@testrail_url}#{@owner}/#{@repo}"
    end

    def compare_branch(head,base)
      response_data = {}
      uri = "/compare/#{head}...#{base}"
      compare_respose = send_request('GET', uri, {})
      response_data[:status] = compare_respose["status"]
      if response_data[:status] != "identical"
        response_data[:commits_behind_by] = "#{response_data[:status]} => #{compare_respose['behind_by']}"
        response_data[:recent_commiter_name] = compare_respose["base_commit"]["commit"]["author"]["name"]
        response_data[:recent_commiter_measage] = compare_respose["base_commit"]["commit"]["message"]
        response_data[:recent_commiter_timstamp] = compare_respose["base_commit"]["commit"]["author"]["date"]
        response_data[:recent_commit_id] = compare_respose["base_commit"]["sha"]
      end
      return response_data
    end

    def create_pull_request(title,body,head,base)
      branch_compare_info = compare_branch(head,base)
      if branch_compare_info[:status] == "identical"
        puts "branch '#{base}' '#{head}' are identical"
      else
        uri = "/pulls"
        data = {
          "title": "#{base} into #{head} - #{branch_compare_info[:commits_behind_by]}",
          "body": "commits_behind - #{branch_compare_info[:commits_behind_by]} \nrecent commit name - #{branch_compare_info[:recent_commiter_name]} \nrecent commit message - #{branch_compare_info[:recent_commiter_measage]} \nrecent commit timestamp - #{branch_compare_info[:recent_commiter_timstamp]} \nrecent commit id - #{branch_compare_info[:recent_commit_id]}",
          "head": head,
          "base": base
        }
        response =  send_request('POST', uri, data)
        puts response["errors"][0]["message"] if response.key?("errors")
        puts response["html_url"] if response.key?("number")
      end
    end

    def send_request(method, uri, data)
      url = URI.parse(@url + uri)
      if method == 'POST'
        request = Net::HTTP::Post.new(url.path)
        request.body = JSON.dump(data)
      else
        request = Net::HTTP::Get.new(url.path)
      end
      request.add_field('Content-Type', 'application/json')
      request.add_field('Authorization', @token)
      conn = Net::HTTP.new(url.host, url.port)
      if url.scheme == 'https'
        conn.use_ssl = true
        conn.verify_mode = OpenSSL::SSL::VERIFY_NONE
      end
      response = conn.request(request)

      if response.body && !response.body.empty?
        result = JSON.parse(response.body)
      else
        result = {}
      end
      return result
    end
  end
end


git_hub_client = Clients::GitHubClient.new
git_hub_client.send(ARGV[0],*[ARGV[1],ARGV[2],ARGV[3],ARGV[4]])
