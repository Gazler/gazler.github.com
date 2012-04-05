---
layout: post
title: "Making Backbone Applications Realtime With Pusher"
date: 2012-04-02 09:40
comments: true
categories:  ["backbone", "pusher", "javascript"]
---

Over the past few months at Powershift, we have been experimenting with the development of Single-Page Applications.  We looked at a few javascript frameworks designed for the task and ultimately decided on [Backbone.js](http://documentcloud.github.com/backbone/)  One of the things we really like about Backbone is that it has a data-driven approach.  That is, that you can listen on an item for a specific trigger, and call an action when that trigger is called.  The most common case for this in our applications is binding a view to the "change" event of a model so that when new data comes in, it is rendered on the page.

{% codeblock lang:js %}
//The View
Application.Views.PostView = Backbone.View.extend({
  initialize: function() {
    this.model.on("change", this.render, this);
  },

  render: function() {
    $(this.el).html(ich.post_template(this.model)); //Render the element
  }
});

//The Model
Application.Models.Post = Backbone.Model.extend({
  url: function() {
    return "/posts/"+this.id;
  });
});

//initialize them
var post = new Application.Models.Post({id: 1});
var postView = new Applications.Views.PostView({model: post});
post.fetch();
post.save({title: "New Title"});
{% endcodeblock %}


<!-- more -->

With this in place, it means that any time our Backbone model is changed, the view will be updated to reflect that.  In this example we have used a Post view and model, but models can represent any resource.  When `post.fetch()` returns, a "change" event is triggered, this event is being listened for on the view, and when it occurs, `render()` in the view is called.  This style of event binding also means that when a save is called, the view is also re-rendered.  This all works well for a single person editing the posts, but what about if we want multiple people to be able to edit the resource, and all others to be notified?

This is where [Pusher](http://pusherapp.com) comes in.  Pusher is a very easy way to add realtime to your applications.  You can get up and running with a few lines of javascript and it is very simple to hook it into Backbone events.

After including the javascript from Pusher (you can get it from their homepage after you sign up) you can create a notifier for all your notifications.  One major benefit of moving everything into one class, is that if for whatever reason, you wish to change the backend used for your notifications (if you for example, write your own one and want to use that instead of Pusher) then all the dependencies are in one place.

{% codeblock lang:js %}
//notifier
Application.Notifier = (function() {

  function Notifier() {
    _.extend(this, Backbone.Events);
    this.pusher = new Pusher(CHANNEL_ID);
    this.channels = {};
  }

  Notifier.prototype.subscribe = function(channel) {
    var self = this;
    this.channels[channel] = this.pusher.subscribe(channel);
    this.channels[channel].bind_all(function(event, data) {
      self.trigger(event, data);
    });
  };

  return Notifier;

})();

//initialize
Application.notifier = new Application.Notifier();
Application.notifier.subscribe("post1");
{% endcodeblock %}

With this in place, any time an event happens on the "post1" channel, a backbone event will be triggered.  We can now listen to this event in the `initialize` function of the model.

{% codeblock lang:js %}
//Application.Models.Post
initialize: function() {
  Application.notifier.on("post:change", this.postChanged, this);
},

postChanged: function() {
  this.fetch(); 
}

{% endcodeblock %}

Unfortunately we can't just call fetch when we receive a "post:change" event as the date received from Pusher is an array, and Backbone is expecting an object.  I did try to [patch this](https://github.com/documentcloud/backbone/pull/1058) but it was not accepted, so we need to wrap the `fetch()` inside another function.


The last thing required to hook this up is to trigger an event when the model is saved.  This depends on your backend however you can also trigger these from the Pusher dashboard on their website after you have signed in, the name of the message you want to send is "post:change".  Using the Pusher gem and Ruby on Rails, you would do:

{% codeblock lang:js %}
Pusher["post1"].trigger!("post:change", "")
{% endcodeblock %}

Everything is now hooked up.  You will notice that if you have your application open in two browsers, when you update one, the other will automatically update.  When a "post:change" event is received from Pusher, the model is listening for it and triggers a `fetch()` which in turn triggers a `render()` on the view.

This is just one way of doing things of course, you can also send data through with your event to prevent performing an additional `fetch()` on the server.  You can also bind the notifier to Backbone Views and Collections and perform any functions you wish when an event is received.
