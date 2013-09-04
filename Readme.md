# Vagrant Chef Server

# Requirements

- virtualbox
- ruby
- Chef
- Vagrant + plugins

    $ vagrant plugin install vagrant-berkshelf
    $ vagrant plugin install vagrant-omnibus

# Install

First clone this repository then 

    $ bundle install --path vendor
    $ berks install --path cookbooks
    $ vagrant up

Visit https://10.33.33.33/version to check if everything is working.
Then https://10.33.33.33 to access the web interface (admin/p@ssw0rd1).
Change to a secure password and regenerate certificates.

# Do it yourself

## Project Setup

Create a move to the project folder.

    $ mkdir vagrant-chef-server
    $ cd vagrant-chef-server
    
Create a gem file.
   
    $ cat > Gemfile <<EOF
    source "https://rubygems.org"

    gem "knife-solo"
    gem "berkshelf"
 
    EOF

Install the gem locally to a new 'vendor' folder.  
    
    $ bundle install --path vendor
    
Create a knife solo (a.k.a chef solo) folder structure.
    
    $ knife solo init .
    
Create a Berkshelf file to manage cookbooks.
    
    $ cat > Berksfile <<EOF
    site :opscode

    cookbook 'chef-server'
    
    EOF

Install Berkshelf cookbooks.
 
    $ berks install --path cookbooks
    
## Configure Chef Server Node

Create “chef.json” file in the role folder with the following content. 

    {
      "name": "chef",
      "chef_type": "role",
      "json_class": "Chef::Role",
      "description": "The base role for Chef Server",
      "default_attributes": {
        "chef-server": {
          "version": "latest",
          "configuration": {
            "chef_server_webui": {
              "enable": true
            }
          }
        }
      },
      "run_list": [
        "recipe[chef-server::default]"
      ]
    }

You can change or find more info to the above settings at https://github.com/opscode-cookbooks/chef-server/blob/master/attributes/default.rb

Done ! 


## Configure Vagrant

Create a vagrant intialization file with the following command.

    vagrant init
    
Change your vagrant to file to:

    # -*- mode: ruby -*-
    # vi: set ft=ruby :

    Vagrant.configure("2") do |config|
      config.vm.box = "precise64"
      config.vm.box_url = "http://files.vagrantup.com/precise64.box"

      config.omnibus.chef_version = :latest
      config.berkshelf.enabled = true
      
      config.vm.define :chef do |chef_server|  
        chef_server.vm.provider :virtualbox do |v|
          v.name = "chef-server"
          #v.customize ["modifyvm", :id, "--cpus", "2"]
          v.customize ["modifyvm", :id, "--memory", "1024"]
        end

        chef_server.vm.network :private_network, ip: "10.33.33.33"

        chef_server.ssh.max_tries = 40
        chef_server.ssh.timeout   = 120

        chef_server.vm.provision :chef_solo do |chef|
          chef.cookbooks_path = "cookbooks"
          chef.roles_path = "roles"
          chef.data_bags_path = "data_bags"
          
          chef.add_role("chef")
          #chef.add_recipe "chef-server::default"
        end
      end

      config.vm.define :chef_client do |chef_client|
        chef_client.vm.provider :virtualbox do |v|
          v.name = "chef-client"
        end
            
        chef_client.vm.network :private_network, ip: "10.33.33.50"

        chef_client.ssh.max_tries = 40
        chef_client.ssh.timeout   = 120
      end
    end

Launch the virtual machines.

    $ vagrant up

Visit https://10.33.33.33/version to check if everything is working.
Then https://10.33.33.33 to access the web interface (admin/p@ssw0rd1).
Change to a secure password and regenerate certificates.


Connect through ssh to copy your chef server keys.

    $ vagrant ssh chef
    vagrant@precise64:~$ sudo cp /etc/chef-server/*.pem /vagrant/.chef/


## Knife configuration

Configure knife

    $ knife configure -i

- chef server URL: https://10.33.33.33
- clientname: admin
- admin client's private key: .chef/webui.pem
- validation key: .chef/validation.pem

Check knife configuration:

    $ knife client list
    > chef-validator
    > chef-webui



