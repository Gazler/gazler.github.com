---
layout: post
title: "OAuth2 Consumer With Sinatra"
date: 2012-01-11 20:40
comments: true
categories: ["rails", "ruby", "sinatra", "oauth"]
---

**This is part 2 of creating an OAuth based API with rails.  [Part 1 is available here](http://blog.gazler.com/blog/2012/01/08/oauth2-provider-with-rails/).**

##Source

The source for both the provider and the consumer are available [here](https://github.com/Gazler/Oauth2-Tutorial)

##Screencasts

I have created screencasts to go along with this tutorial.  This is my first attempt at screencasting, so please drop me a message if you find them useful or if there is anything you think can be improved.  Your feedback is appreciated.

*Download* [mp4 format](http://screencasts.gazler.com/provider.mp4) [ogv format](http://screencasts.gazler.com/provider.ogv) [avi format](http://screencasts.gazler.com/provider.avi)

Change the following in *views/oauth/oauth2_authorize.html.erb*

{% codeblock lang:ruby %}
<p>Would you like to authorize <%= link_to @token.client_application.name,@token.client_application.url %> (<%= link_to @token.client_application.url,@token.client_application.url %>) to access your account?</p>
{% endcodeblock %}
To

{% codeblock views/oauth/oauth2_authorize.html.erb lang:ruby %}
<p>Would you like to authorize <%= link_to @client_application.name,@client_application.url %> (<%= link_to @client_application.url,@client_application.url %>) to access your account?</p>
{% endcodeblock %}
    
<!-- more -->

You should now start a rails server and navigate to http://localhost:3000/users/sign_up, after signing up go to http://localhost:3000/oauth_clients and create a client.  Please not that your client callback_url must match that of the one passed through in your app.  If you are using the demo sinatra app, it should be **http://localhost:4567/auth/test**

There are a couple things you should change in *views/oauth_clients/index.html.erb* Change the @tokens block to:

{% codeblock views/oauth_clients/index.html.erb lang:ruby %}
 <% @tokens.each do |token|%>
  <tr>
    <td><%= link_to token.client_application.name, token.client_application.url %></td>
    <td><%= token.authorized_at %></td>
    <td>&nbsp;</td>
  </tr>
<% end %>
{% endcodeblock %}

And change the @client_applications block to:

{% codeblock views/oauth_clients/index.html.erb lang:ruby %}
<% @client_applications.each do |client|%>
  <div>
    <%= link_to client.name, oauth_client_path(client) %>-
      <%= link_to 'Edit', edit_oauth_client_path(client) %>
      <%= link_to 'Delete', oauth_client_path(client), :confirm => "Are you sure?", :method => :delete %>
  </div>
<% end %>
{% endcodeblock %}

You should now create a consumer directory outside of the rails root.

    cd ..
    mkdir consumer && cd consumer
    
You will then need to install sinatra and the oauth2 gem **Please note this requires a version of oauth higher than 0.5 **
    gem install sinatra
    gem install oauth2 

Copy the following code, replacing the API keys from those of the client:


{% codeblock app.rb lang:ruby %}
require 'sinatra'
require 'oauth2'
require 'json'
enable :sessions

def client
  OAuth2::Client.new(consumer_key, consumer_secret, :site => "http://localhost:3000")
end

get "/auth/test" do
  redirect client.auth_code.authorize_url(:redirect_uri => redirect_uri)
end

get '/auth/test/callback' do
  access_token = client.auth_code.get_token(params[:code], :redirect_uri => redirect_uri)
  session[:access_token] = access_token.token
  @message = "Successfully authenticated with the server"
  erb :success
end

get '/yet_another' do
  @message = get_response('data.json')
  erb :success
end
get '/another_page' do
  @message = get_response('data.json')
  erb :another
end

def get_response(url)
  access_token = OAuth2::AccessToken.new(client, session[:access_token])
  JSON.parse(access_token.get("/api/v1/#{url}").body)
end


def redirect_uri
  uri = URI.parse(request.url)
  uri.path = '/auth/test/callback'
  uri.query = nil
  uri.to_s
end
{% endcodeblock %}
    
You can grab the required views from *consumer/views*
