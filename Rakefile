#!/usr/bin/env ruby
require "rake"

## methods ################################################

# executes shell command w/login profile
def bash cmd
  %x{ bash --login -c "#{ cmd }" }
end

## tasks ##################################################

namespace :ansible do
  desc "shows hiera for host"
  task :facts, [ :host ] do | t, arguments |
    puts bash( "./bin/extends --facts=#{ arguments[:host] } ansible -i/etc/ansible/hosts" )
  end

  desc "shows groups that host belongs to"
  task :groups, [ :host ] do | t, arguments |
    puts bash( "./bin/extends --host=#{ arguments[:host] } ansible -i/etc/ansible/hosts" )
  end

  desc "shows hosts that belong to group"
  task :hosts, [ :group ] do | t, arguments |
    puts bash( "./bin/extends --group=#{ arguments[:group] } ansible -i/etc/ansible/hosts" )
  end

  desc "generates docker nodes from inventory file"
  task :nodes, [ :inventory ] do | t, arguments |
    port = 2221
    content = File.read arguments[:inventory]
    matches = ( content.scan /^.+?ansible_host.+$/ )

    %x{
      docker network rm ansible.gwambia
      docker network create -d bridge --subnet 172.25.0.0/16 ansible.gwambia
    }

    # first we iterate through matches so that we can get host
    # name and address
    hosts = matches.map do | line |
      {
        name: ( line.match /^(?<name>.+?)\s/ )['name'],
        address: ( line.match r = /ansible_host\s*?=\s*?(?<host>.+?)(\s|$)/ )['host'],
      }
    end
    hosts.each do | host |
      # replace host with loopback address and port
      content.gsub! /ansible_host=#{ host[:address] }/, " ansible_host=127.0.0.1 ansible_port=#{ port += 1 } "

      # determine ubuntu version based upon group membership
      groups = `rake ansible:groups[#{ host[:name] }]`.split "\n"
      tag = groups.include?( "xenial" ) && "16.04" || "14.04.5"

      # launch docker container
      fork do
        exec %{
          docker rm -f #{ name } > /dev/null 2>&1
          docker run \
            -it \
            -d \
            --name #{ name } \
            --publish #{ port }:22 \
            --volume $SSH_AUTH_SOCK:/ssh-agent \
            --env SSH_AUTH_SOCK="$SSH_AUTH_SOCK" \
            --network ansible.gwambia \
            ubuntu:#{ tag } /bin/bash -l

          docker exec #{ name } bash -c "
            apt update
            apt install -y \
              sudo vim ssh telnet net-tools python-minimal
            service ssh start
          "

          pub=`cat ~/.ssh/id_rsa.pub`
          docker exec #{ name } bash -c "
            mkdir ~/.ssh; echo '$pub' >> ~/.ssh/authorized_keys
          "
        }
      end
    end
    Process.wait

    puts content
    exit

    # finally write inventory.development
    File.write "#{ arguments[:inventory] }.development", content
  end
end
