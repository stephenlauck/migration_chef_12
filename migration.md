# Migrating from Chef 10.x to 12.x
This document was originally created for a migration from Chef 10.  However, this guide works just as we for a migration from Chef 11.x to Chef 12.x.  Wherever you see Chef 10 you can replace with Chef 11.

## Overview:
The steps below follow the documentation provided on docs.opscode.com, but also leverage git as a way to manage multiple knife.rb configurations in your .chef directory, and uses the chef-client cookbook to rollout the chef_server_url attribute to the client.rb file on your managed nodes.

## Before you begin:
###You should already have the following setup:
* An Chef 10.x server already set up and managing nodes
* An Chef 12.x server configured with access to network that the 10.x server is already managing
* A Chef Workstation with Chefdk and git installed with access to both your 10.x server as well as the 12.x server
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

Create a new git branch with the name of your current chef server:

`git checkout -b 10x_server`

Copy your .chef dir for the 10.x server credentials into the chef_migration dir
`cp -r ~/PATHTOYOUR/.chef .`

Test and verify the credentials. You should see ORGNAME-validator in the list.

`knife client list`

Check in your changes to the current git branch:

```
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

Download all of the data from your Open Source Chef Server, make sure you are in ~/chef_migration

`knife download / --chef-repo-path .`

Commit the download to the branch

```
git add .
git commit -m “download of Chef 10.x”
```

Create a new git branch for the Enterprise Chef Server configurations:

$ git checkout -b 12x_server

Copy your .chef dir for the 12.x server credentials into the chef_migration dir
`cp -r ~/PATHTOYOUR/.chef .`

Test and verify the credentials. You should see ORGNAME-validator in the list.

`knife client list`

Check in your changes to the knife.rb to the current git branch:

```
git add .
git commit -m “add credentials for Chef 12.x”
```

Upload the data from your 10.x server to the 12.x:

`knife upload / --chef-repo-path .`

Warnings or errors may occur if
* org names differ

Create the following script and run it on the Chef server to correctly set permissions for clients

```
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
```

Execute the script on your 12.x server

`knife exec fix_permissions.rb`

Switch back to your 10x git branch:

`git checkout 10x_server`


Add the [chef-client::config](https://github.com/opscode-cookbooks/chef-client#usage) recipe to the nodes you want to migrate

Update the chef_server_url attribute in an environment file to point to the 12.x server

```
chef_client:
  config:
    chef_server_url: https://<12xServerURL>/organizations/ORGNAME
```

Initiate a chef-client run on a test set of nodes to pick up the change to point to the 12.x server and debug any issues

You should now see your nodes checking in on the 12.x server.

Notes:

https://gist.github.com/irvingpop/6c545a335febc81ac950
