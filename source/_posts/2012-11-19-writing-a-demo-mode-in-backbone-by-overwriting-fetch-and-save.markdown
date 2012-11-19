---
layout: post
title: "Writing a Demo Mode in Backbone"
date: 2012-11-19 18:59
comments: true
categories:  ["backbone.js", "javascript"]
---

##Background

When writing an application in Backbone, it is highly likely that you utilize [fetch()](http://backbonejs.org/#Collection-fetch) to retrieve data from your web server and [save()](http://backbonejs.org/#Model-save) to store data on it.  This is exactly what we did for [Uptilt](http://upti.lt) our entry for the Rails Rumble 2012 (the making of you can read about on [http://uptiltgame.tumblr.com/](the Uptilt Blog)).

One of the key issues with Uptilt being a two player game is that it needs two people to play the game.  The solution to this is to use a demo (or practice) mode.  In the case of Uptilt, this involved implementing the game rules for card comparison client-side.  We also didn't want any interaction with the server when the practice game is in progress as the game is not recorded in the database.

<!-- more -->

##Overwriting fetch() and save()

As is the case in most Backbone applications, we have an init function defined in the namespace which starts everything off (sets up the collections and renders the first views.)


{% codeblock lang:js %}
$(function() {
  uptilt.init({
    game_id: <%=@game.id%>,
    //Lots of other data
    timer: 30
  });
});
{% endcodeblock %}

This is the only code needed from the HTML to pass all the data through to the Backbone application and get things started.  The init function looks something like:


{% codeblock lang:js %}
init: function(options) {
  _.bindAll(this);
  _.extend(this, Backbone.Events);

  this.first_card = new Uptilt.Models.Card(options.first_card);
  this.player = new Uptilt.Models.Player(options.player);
  this.opponent = new Uptilt.Models.Player(options.opponent);

  window.notifier.subscribe(options.channel);
  window.notifier.on("connected", this.startGame);
},
{% endcodeblock %}

The important points are that:

 * There are some variables that are set to some values passed through from the HTML call to the init function
 * The notifier (which I have [posted about before](http://blog.gazler.com/blog/2012/04/02/making-backbone-applications-realtime-with-pusher/) will trigger the preload function when connected is called.

Here is the startGame function:

{% codeblock lang:js %}
startGame: function() {
  this.board = new Uptilt.Views.Board();
  $("#main").append(this.board.render().el);
}
{% endcodeblock %}

The specifics about the internals are unimportant except for the following points:

 * When a user makes a turn, a `save()` is called on an instance of Uptilt.Models.Turn
 * When a new card is required a `fetch()` is called on an instance of Uptilt.Models.card and the server sends a new card to the player

 Since these are the only two interactions with the server, the game could be implemented client-side by storing the state of the game on the client (since this is a one-player mode, cheating isn't an issue, you are only cheating yourself!)


 The first step in doing so is to call a new method instead of init when the page loads:

{% codeblock lang:js %}
Uptilt.practice({
  cards: //cards JSON (of 30 cards)
  player: //player JSON
  opponent: //opponent JSON
});
{% endcodeblock %}

The practice code looks like this:

{% codeblock lang:js %}
  practice: function(options) {
    window.notifier.subscribe = function() {};
    this.cards = options.cards;
    [this.playerCards, this.opponentCards] = this.shuffleCards();
    options.first_card = this.playerCards[0];
    this.practiceModels();
    this.init(options);
    this.startGame();
  },
{% endcodeblock %}

You will notice that this code is responsible for:

 * Stubbing out the notifier (No push notifications are required in a single player game)
 * Setting up the initial card state
 * Overriding the first_card variable that init expects.
 * Calling init
 * Manually calling startGame (Since the connected message on the notifier will never trigger)

 The most important method here is the practiceModels method, which looks like:

 {% codeblock lang:js %}
practiceModels: function() {
  var self = this;
  Uptilt.Models.Turn = Backbone.Model.extend({
    save: function() {
      self.determineOutcome(this.get("move").rule_key);
    }
  });

  Uptilt.Models.Card = Backbone.Model.extend({
    fetch: function() {
      this.attributes = self.playerCards[0];
      this.trigger("change");
    }
  });
},
 {% endcodeblock %}

 This function overrides the save method of Turn (that is what is called when a user clicks on their chosen attribute in the game of Uptilt) and instead calls determineOutcome (which is not shown but it is where the game logic is called client-side) emulating the code that is called on the server in a "real" game of Uptilt.

 It also overrides the fetch method of Card.  In Uptilt, a player can only see their current card to prevent a user being able to work out the cards that the opponent has in their hand.  In the case of the client, this is simply the card at the top of the players pile.  Triggering the "change" event is important as that is what the views are listening to and they will re-render accordingly.

##The Results

That is all I had to do to make the game playable on the client side.  The entire commit was about 100 lines of JavaScript and not a single view had to be modified.  This same technique could be used to implement a tutorial style introduction to your site with ease.

Does it work?  Why not try out a [practice game](http://upti.lt/decks/3/practice) of Uptilt and see for yourself.
