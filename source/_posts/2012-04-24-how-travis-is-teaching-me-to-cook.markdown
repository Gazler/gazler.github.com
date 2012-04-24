---
layout: post
title: "How Travis Is Teaching Me To Cook"
date: 2012-04-24 12:39
comments: true
categories:  ["ruby", "chef", "vagrant", "rails"]
---

**Note: the contents of this post are heavily influenced by [Sous Chef](https://github.com/michaelklishin/sous-chef). **

Over the past couple of weeks I have been working with a mobile developer to help integrate with one of my applications.  We plan on using OAuth (Which I have blogged about [here](http://blog.gazler.com/blog/2012/01/08/oauth2-provider-with-rails/)) to authenticate the mobile application with the server.  The problem with this is that the rails application is still in development, and the mobile developer doesn't have the code running.  I don't want them to worry about dependencies, setting up the database or setting up git access via ssh keys, I just want him to be able to run the server.

This is where [Vagrant](http://vagrantup.com/) comes in.  Vagrant allows you to create Virtual Machines (using VirtualBox) programatically.  This means that I can create a VM of my application, hand it to anothe developer and let them get on with it.  Vagrant can utilize [Chef](http://www.opscode.com/chef/) to build the machine.  Chef is an integration framework that makes it easy to deploy servers, which it does using cookbooks and recipes.  A recipe is for installing a particular piece of software and a cookbook is a collection of recipes.  There are plenty of cookbooks available online, I opted to use a set of cookbooks developed by [Travis CI](http://travis-ci.org) that are available [on GitHub](https://github.com/travis-ci/travis-cookbooks)

##Getting Started

The first thing to do is install Vagrant.  I chose to use the gem by doing:

    gem install vagrant

However there are binaries available on their [downloads](http://downloads.vagrantup.com/tags/v1.0.2) page.

<!-- more -->

The next step is to create a folder for vagrant to sit in and intialize vagrant.

    mkdir my_application && cd my_application
    vagrant init

This will create a Vagrantfile in your repository that should look like this:

{% codeblock Vagrantfile lang:ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant::Config.run do |config|
  config.vm.box = "base"

  #lots of comments!
end
{% endcodeblock %}

The first two lines tell your editor what syntax highlighting to use, which is ruby.  We're going to make some changes inside the config block, so remove the `config.vm.box = "base"` line.  The next step is to specify the "box" that is, the machine we want to boot from.  In this case I am using Ubuntu 11.04 provided by Travis CI.

{% codeblock Vagrantfile lang:ruby %}
Vagrant::Config.run do |config|

   config.vm.box     = "natty32_base"
   config.vm.box_url = "http://files.travis-ci.org/boxes/bases/natty32.box"

end
{% endcodeblock %}

This is all you need to start up.  If you run the command:

    vagrant up

Then the virtual box will be started, and you can shell into it using `vagrant ssh`.

##Cooking With Chef

Now that the VM is working, you could ssh in and install all the dependencies, but that isn't really a good way of doing it.  You would have to provide build instructions to other people using the box.  Instead, Chef can be used to install software on the VM.

First of all, clone the cookbooks from Travis CI (or somewhere else if you use other cookbooks.)

    git clone git://github.com/travis-ci/travis-cookbooks.git cookbooks

With the cookbooks, you can now modify your Vagrantfile to install software.


{% codeblock Vagrantfile lang:ruby %}
Vagrant::Config.run do |config|

    config.vm.box     = "natty32_base"
    config.vm.box_url = "http://files.travis-ci.org/boxes/bases/natty32.box"

    config.vm.provision :chef_solo do |chef|
      chef.cookbooks_path = ["cookbooks/ci_environment"]
      chef.log_level = :debug


      #Install application dependencies - apt and build-essential are a must
      chef.add_recipe "apt"
      chef.add_recipe "build-essential"
      chef.add_recipe "git"
      chef.add_recipe "rvm"
      chef.add_recipe "sqlite"
      chef.json.merge!({
                          :apt => {
                            :mirror => :ru
                          },
                          :rvm => {
                            :rubies  => [{ :name => "1.9.2" }]
                          },
                          :travis_build_environment => {
                            :user => "vagrant",
                            :group => "vagrant"
                          }
                       })
    end
end
{% endcodeblock %}

You can call `chef.add_recipe` to install all of your application dependencies, you can check the cookbooks/ci_environment folder to see which recipes are available.  The last line in the config (`chef.json.merge`) contains the configuration used by the recipes ([read more here](http://wiki.opscode.com/display/chef/Attributes).)


If you run `vagrant provision` then it will install all your dependencies (including ruby 1.9.2 as specified in the json block.)

When this finishes, you can ssh into your box by calling `vagrant ssh` and check that ruby has been installed by typing `ruby -v` and it should return 1.9.2

##Running your application

It is now time to add the application to the box.  To do this, a new recipe must be created.  I would recommend creating a new folder for your cookbook.  I called mine "application_cookbook"


    mkdir -p application_cookbook/main/recipes

"main" is the name of your recipe and "recipes" is where the recipes live.  Create a new file in recipes called "default.rb" and inside the file, put the following:

{% codeblock default.rb %}
application_dir = "/home/vagrant/application"
#install bundler
gem_package "bundler"

#update rubygems
execute "gem" do
  command "gem update --system"
  cwd application_dir 
end

execute "bundle" do
  command "bundle"
  cwd application_dir 
end

#clear and migrate the database
execute "bundle" do
  command "bundle exec rake db:drop db:migrate db:seed"
  cwd application_dir 
end

#start the server and daemonize it
execute "rails" do
  command "rails s -d"
  cwd application_dir 
end
{% endcodeblock %}

This will install all the required gems and set up the database.  You may notice that we reference /home/vagrant/applicaiton, this is a folder we haven't created yet.  To do so we need some additional changes to the Vagrantfile.

{% codeblock Vagrantfile lang:ruby %}
Vagrant::Config.run do |config|
  config.vm.provision :chef_solo do |chef|
    #Change the cookbooks_path to inclide your new recipes
    chef.cookbooks_path = ["cookbooks/ci_environment", "application_cookbook"]

    #keep all the old code here
    
    #Add your new recipe
    chef.add_recipe "main"
  end
  
  #Create a reference to your application folder
  config.vm.share_folder "application", "application", "application", :create => true
  #forward requests for port 3000 to the virtual box
  config.vm.forward_port 3000, 3000
end
{% endcodeblock %}

The final step is to move your application folder into your current directory `cp -r /path/to/application .`

You will need to reload vagrant as there are port and share folder changes.

    vagrant reload

When everything finishes, when you navigate to [http://localhost:3000](http://localhost:3000) then you should see your application!


After I followed the above steps, I could zip up the folder, give it to the other developer and he was set.  He could extract the files, run `vagrant up` and access the server.
