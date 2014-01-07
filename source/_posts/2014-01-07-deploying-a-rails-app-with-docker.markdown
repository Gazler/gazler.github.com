---
layout: post
title: "Deploying a Rails App With Docker"
date: 2014-01-07 20:27
comments: true
published: false
categories: ["rails", "postgresql", "redis", "docker", "nginx"]
---

##The Structure

This post will highlight a process that can be used to deploy rails application using a series of docker containers - capistrano will be used for deployment. We will put these containers all on the same server, but there is no reason that these containers couldn't all exist on separate servers. When the containers are running there will be the following:

 * The application - a sample Ruby on Rails app that connects to PostgreSQL and redis.
 * A PostgreSQL server - the database (PostgreSQL 9.2) server, we will use the [Ambassador Pattern](http://docs.docker.io/en/latest/use/ambassador_pattern_linking/) to make it easier to move the database to a different server if required.
 * A redis server - A redis server, we will also use the ambassador pattern here.
 * A load balancer - This will use nginx as a reverse proxy to communicate with the rails app - this will be the only server that exposes a port (port 80) we will use a [docker link](http://docs.docker.io/en/latest/use/working_with_links_names/) for this.


##The Rails App

Generate a new rails app with PostgreSQL as the database and add redis as a dependency:

    rails new sample -d postgresql
    cd sample
    echo "gem 'redis'" >> Gemfile
    bundle

Generate a model that can be used to prove the database works:

    rails g scaffold user name

For demonstrative purposes create some users in the migration:


{% codeblock db/migrate/xxxx_create_users.rb lang:ruby %}
class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t| 
      t.string :name
      t.timestamps
    end 

    reversible do |dir|
      dir.up do
        User.create(:name => "First User")
        User.create(:name => "Second User")
        User.create(:name => "Third User")
      end 
    end 

  end 
end
{% endcodeblock %}    

To test that redis works we can modify the view for the index action:


{% codeblock db/migrate/xxxx_create_users.rb lang:ruby %}
<h1>Listing users</h1>
<% redis = Redis.new(:host => ENV["REDIS_PORT_6379_TCP_ADDR"], :port => ENV["REDIS_PORT_6379_TCP_PORT"]) %>
<%= redis.incr("counter") %>
<table>
{% endcodeblock %}

There are a couple of environment variables here - these variables are created by docker when we use the "-link" argument. This leads us nicely on to the redis container.

##The Redis Container

We can use an existing image from the [docker index](https://index.docker.io)

    docker run -d -name redis crosbymichael/redis

We will also create an ambassador so that our application doesn't connect directly to the redis container.

    docker run -d -link redis:redis -name redis_ambassador -p 6379:6379 svendowideit/ambassador

##The PostgreSQL Container

We can also use an existing image for this:

    docker run -d -name postgresql zumbrunnen/postgresql

And an ambassador:

    docker run -d -link postgresql:postgresql -name postgresql_ambassador -p 5432:5432 svendowideit/ambassador

##The Application Container

With these things in place, we are ready to build the container for the application.
