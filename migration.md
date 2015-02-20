# Migrating from Chef 10.x to 12.x

## Overview:
The steps below follow the documentation provided on docs.opscode.com, but also leverage git as a way to manage multiple knife.rb configurations in your .chef directory, and uses the chef-client cookbook to rollout the chef_server_url attribute to the client.rb file on your managed nodes.

## Before you begin:
###You should already have the following setup:
* An Chef 10.x server already set up and managing nodes
* An Chef 12.x server configured with access to network that the 10.x server is already managing
* A Chef Workstation with Chefdk adn git installed with access to both your 10.x server as well as the 12.x server
* The [chef-client](https://github.com/opscode-cookbooks/chef-client) cookbook running on the nodes you are managing

Create a new folder where you intend to do your migration:

```
mkdir ~/chef_migration
cd chef_migration
```

Create a .chef for Chef credentials

`mkdir .chef`

Initialize a new git repository and commit a new master:

```
git init
git add .
git commit -m “Initial commit”
```

Create a new git branch with the name of your open source chef server:

`git checkout -b 10x_server`

Copy your .chef dir for the 10.x server credentials into the chef_migration dir
`cp -r ~/PATHTOYOUR/.chef .`

Test and verify the credentials. You should see ORGNAME-validator in the list.

`knife client list`

Check in your changes to the current git branch:

```
git init
git add .
git commit -m “add credentials for Chef 10.x”
```

Add versioned cookbooks to the knife.rb to download all versions of your Chef cookbooks in case previous versions have been locked in environments.

`emacs .chef/knife.rb`

```
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                "customer"
client_key               "#{current_dir}/customer.pem"
validation_client_name   "customer-org-validator"
validation_key           "#{current_dir}/customer-org-validator.pem"
chef_server_url          "https://api.opscode.com/organizations/customer-org"
syntax_check_cache_path  "#{ENV['HOME']}/.chef/syntaxcache"
cookbook_path            ["#{current_dir}/../cookbooks"]

versioned_cookbooks true
```

Download all of the data from your Open Source Chef Server:

`knife download / --chef-repo-path ~/chef_migration/`

Create a new git branch for the Enterprise Chef Server configurations:

$ git checkout -b ENTERPRISE_CHEF_SERVER


Setup your knife.rb and .pem file with access to the Enterprise:

$ cat /path/to/ec/.chef/knife.rb > $ cat ~/chef_migration/.chef/knife.rb
$ cat /path/to/ec/.chef/USER.pem > $ cat ~/chef_migration/.chef/USER.pem


Check in your changes to the knife.rb to the current git branch:

$ git init
$ git add .
$ git commit -m “Initial commit to ENTERPRISE_CHEF_SERVER”


Upload the data from your OSC server to the EC:

$ knife upload / --chef-repo-path ~/chef_migration/downloads


Create the following script ~/chef_migration/fix_permissions.rb

#!/usr/bin/env ruby
require 'rubygems'
require 'chef/knife'

Chef::Config.from_file(File.join(Chef::Knife.chef_config_dir, 'knife.rb'))
rest = Chef::REST.new(Chef::Config[:chef_server_url])

Chef::Node.list.each do |node|
  %w{read update delete grant}.each do |perm|
    ace = rest.get("nodes/#{node[0]}/_acl")[perm]
    ace['actors'] << node[0] unless ace['actors'].include?(node[0])
    rest.put("nodes/#{node[0]}/_acl/#{perm}", perm => ace)
    puts "Client \"#{node[0]}\" granted \"#{perm}\" access on node \"#{node[0]}\""
  end
end
   The above code snippet was taken from http://docs.opscode.com/chef/migrate_to_hosted.html



Execute the script on your Enterprise Chef Server:

$ knife exec fix_permissions.rb


Switch back to your OSC git branch:

$ git checkout OPEN_SOURCE_CHEF_SERVER


Update the chef_server_url attribute in your environment file to point to the new EC

chef_client:
  config:
    chef_server_url: https://<EC.DOMAIN.COM/organizations/ORGNAME


Initiate a chef-client run on your nodes to pick up the change to point to the EC

$ sudo chef-client


You should now see your nodes checking in. Login to the Enterprise Chef server
