---
layout: post
title: "OAuth2 Provider With Rails"
date: 2012-01-08 21:16
comments: true
categories:  ["rails", "ruby", "oauth"]
---


Recently I had the need to create an Oauth2 authenticated API.  The following is an app in its most simple form to get you started with creating and testing an Oauth2 powered API, using oauth-plugin, devise and rspec.

##Screencasts

I have created screencasts to go along with this tutorial.  This is my first attempt at screencasting, so please drop me a message if you find them useful or if there is anything you think can be improved.  Your feedback is appreciated.

*Download* [mp4 format](http://screencasts.gazler.com/provider.mp4) [ogv format](http://screencasts.gazler.com/provider.ogv) [avi format](http://screencasts.gazler.com/provider.avi)

##Creating The Provilder

Start by opening up your terminal.  For demonstration purposes I recommend creating a folder called oauth to put both the provider and consumer.

    mkdir oauth && cd oauth
    rails new provider
    cd provider
    
The next step is to add the oauth-plugin gem to your Gemfile.  For this demo I will also be using devise for authentication.  If you wish to use RSpec as your testing framework, now would be the time to add it.

{% codeblock Gemfile lang:ruby %}
gem 'devise'
gem "oauth-plugin", ">= 0.4.0.rc2"
group :test do
    gem 'rspec-rails'
end
{% endcodeblock %}    

You should run *bundle install* to install the oauth-plugin (and rspec.)

If you are using rspec then run:

    rails g rspec:install

If you are using devise then you should create your devise install and user.

    rails generate devise:install
    rails generate devise User
    
Then create the oauth provider (Note I am using rspec)

    rails g oauth_provider --test-framework=rspec
    
And migrate the database
    
    rake db:migrate
    
Might as well do the test database here too

    rake db:test:prepare
    
This will generate some files, there are a few changes required for everything to work.  The first is to delete the file *spec/controllers/oauth_clients_controller_spec.rb* as mentioned in [this commit](https://github.com/pelle/oauth-plugin/commit/6e24ec0ee2f3dc871756b2e8a75fa2181ff504f4).  You should also remove */spec/models/oauth_token_spec.rb* as we are dealing exclusively with oauth2.
The second change is in your *config/routes.rb* file, add:

{% codeblock config/routes/rb lang:ruby %}
root :to => "oauth_clients#index"
{% endcodeblock %} 

    
You will also need to add the following methods to your *app/controllers/application_controller.rb* to make things work as the oauth-plugin gem required a current_user= method.

{% codeblock app/controllers/application_controller.rb lang:ruby %}
def current_user=(user)
  current_user = user
end
{% endcodeblock %} 

You need to add the following to your user model:

{% codeblock app/models/user.rb lang:ruby %}
has_many :client_applications
has_many :tokens, :class_name=>"Oauth2Token",:order=>"authorized_at desc",:include=>[:client_application]
{% endcodeblock %} 
    
You need to add the following attr_accessor to *app/models/oauth_token.rb*

{% codeblock app/models/oauth_token.rb lang:ruby %}
attr_accessor :expires_at
{% endcodeblock %} 

The following alias to *app/controllers/oauth_controller.rb* **and** *app/controllers/oauth_clients_controller.rb*

{% codeblock app/controllers/oauth_controller.rb|app/controllers/oauth_clients_controller.rb lang:ruby %}
alias :login_required :authenticate_user!
{% endcodeblock %} 
    
And finally add the following to *config/application.rb*

{% codeblock config/application.rb lang:ruby %}
require 'oauth/rack/oauth_filter'
config.middleware.use OAuth::Rack::OAuthFilter
{% endcodeblock %} 
    
For the purposes of this test, we will use fixtures, I recommend using factories for real testing.  Grab the 4 fixtures files out of *spec/fixtures* (I got them from the oauth-plugin but they were not included in the generator)

After these files are included, you can run rspec to test what we have so far.

    bundle exec rspec spec
    
There should be 23 examples, all passing.

You should now create a basic rspec test for what will be your API call.  Grab my one out of *spec/api/v1/data_controller_spec.rb*  Also copy the file *support/api_helper.rb*

When you run rspec on this, it should error, you now need to create your API controller.  Since in this example all the API calls will require a valid oauth token, let's create a base controller and then our data controller.

    rails g controller API::V1::Base
    rails g controller API::V1::Data
    
Change the DataController so it extends API::V1::BaseController

{% codeblock app/controllers/api/v1/data_controller.rb lang:ruby %}
class Api::V1::DataController < Api::V1::BaseController
end
{% endcodeblock %} 
    
Now create the routes, add the following to your *config/routes.rb* file

{% codeblock config/routes.rb lang:ruby %}
namespace :api do
  namespace :v1 do
    match "data" => "data#show"
  end
end
{% endcodeblock %} 
    
You will need a show action in your data controller (*app/controllers/api/v1/data_controller.rb*)

{% codeblock app/controllers/api/v1/data_controller.rb lang:ruby %}
def show
  respond_with ({:super_secret => "oauth_data"})
end
{% endcodeblock %} 
    
You will also need to specify the formats that your controllers responds to in your base controller (*app/controllers/api/v1/base_controller.rb*)

{% codeblock app/controllers/api/v1/base_controller.rb lang:ruby %}
respond_to :json, :xml
{% endcodeblock %} 
    
You should also specify which methods require oauth, since it is all in this case, also add the following to your base controllers (the interactive flag is the equivalant of oauth_or_login_required, we want oauth only so we disable it. 

{% codeblock app/controllers/api/v1/base_controller.rb lang:ruby %}
oauthenticate :interactive=>false
{% endcodeblock %} 

If we run our test specs again now, they should pass and there you have it, the beginnings of an Oauth2 API.

Stay tuned for part 2, The Consumer.
