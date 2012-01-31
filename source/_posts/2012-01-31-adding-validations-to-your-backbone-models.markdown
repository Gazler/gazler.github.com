---
layout: post
title: "Adding Validations To Your Backbone Models"
date: 2012-01-31 21:42
comments: true
categories: ["backbone.js", "jasmine", "javascript"] 
---

[Backbone.js](http://documentcloud.github.com/backbone/) is a JavaScript framework that gives your code a much needed structure.  This post will advise you on how to extend the framework by adding in validators.  I will assume you have atleast a basic knowledge of Backbone.

##Source

The source code for this post is [available on gitub.](http://github.com/Gazler/backbone_validators)


##The Goal
While Backbone does give your project structure, it does not give you the full feature set that you require to build a web application.  Many methods are deliberately stubbed out and you are encouraged to fill in the blanks to give you the desired functionality.  One such method is the [validate](http://documentcloud.github.com/backbone/#Model-validate) method on a Backbone Model.

The validate method is called every time `set` or `save` is called on a model.  The aim here is to override the validate method to use an object defined on the model and use this for validation.  We will also add an errors hash to the model which can be displayed to the user.


<!-- more -->

##Preperations

Validations are perfect for unit testing, you have an input and a desired output and not much in between.  I will be using Jasmine for this purpose.  The first step is to [download the spec runner](http://pivotal.github.com/jasmine/download.html).  You can then extract it and locate the SpecRunner.

After downloading, delete the two files `PlayerSpec.js` and `SpecHelper.js` and create a new file `validator_spec.js`.  Then go into the src folder and delete `Player.js` and `Song.js`, you should also create a file `backbone_validator.js`.

We will also need [Backbone.js (0.9.0)](http://documentcloud.github.com/backbone/) and [Underscore.js](http://documentcloud.github.com/underscore/), put these files inside the `lib` folder.

**Optionally** you should include some internationalization of some sort for the error messages.  I strongly suggest [i18n-js](https://github.com/fnando/i18n-js).  You can download the required javascript [here](https://github.com/fnando/i18n-js/blob/master/vendor/assets/javascripts/i18n.js).

Finally, the file `SpecRunner.html` needs to be modified to reference the files you created. It should look like the following:

{% codeblock SpecRunner.html %}
<link rel="shortcut icon" type="image/png" href="lib/jasmine-1.1.0.rc1/jasmine_favicon.png">

<link rel="stylesheet" type="text/css" href="lib/jasmine-1.1.0.rc1/jasmine.css">
<script type="text/javascript" src="lib/jasmine-1.1.0.rc1/jasmine.js"></script>
<script type="text/javascript" src="lib/jasmine-1.1.0.rc1/jasmine-html.js"></script>
<script type="text/javascript" src="lib/i18n.js"></script>
<script type="text/javascript" src="lib/underscore.js"></script>
<script type="text/javascript" src="lib/backbone.js"></script>

<!-- include source files here... -->
<script type="text/javascript" src="src/backbone_validator.js"></script>

<!-- include spec files here... -->
<script type="text/javascript" src="spec/validator_spec.js"></script>
{% endcodeblock %}

##Getting Started

If you have read any Backbone tutorials, most will recommend that you namespace your application.  For this application, I am going to use `Gazler` as the namespace, you can pick whatever you like.


{% codeblock src/backbone_validator.js %}
I18n.translations = {};
I18n.translations["en"] = {
  "errors": {
    "form": {
      "required": "is required"
    }
  }
}

var Gazler = {};

Gazler.Model = Backbone.Model.extend({
  validates: {},
  errors: {},

  validate: function(changedAttributes) {
    this.errors = Backbone.Validate(this, changedAttributes);
    if (!_.isEmpty(this.errors)) {
      return this.errors;
    }
  }
});

Backbone.Validate = function(model, changedAttributes) {
};
{% endcodeblock %}

A few things of not here:

 * We have bootstapped a model here, this is all we will have to touch on Gazler.Model
 * Validate only returns if there are errors, Backbone will not continue with the save action if validate returns **anything**
 * We have added an empty validates hash, this will be where we store our rules.
 * We have added some strings at the top, normally these would be generated from the language file you are using (**config/locales** in ruby)


With everything ready to go, we can write our first test.

{% codeblock validator_spec.js %}
describe("Model Validations", function() {

  beforeEach(function() {
    //instantiate a model
    this.model = new Gazler.Model();
    //A Url property must be specified
    this.model.url = "/";
  });

  describe("required", function() {
  
    it("should add an error when a required field is blank", function() {
      this.model.validates = {
        required: ["name"]
      };

      this.model.validate();
      expect(this.model.errors).toEqual({"name": ["is required"]});
    });
    
  });
  
});
{% endcodeblock %}  


This should be pretty self explanatory, the key part here is that we add the validates hash to the model before each test and then call validate. We then simply check the errors hash against what we expect it to output.  If you open `SpecHelper.html` then you should see that this test is failing.

To make this test pass, we need to run through the rules specified in the validates hash and see if any of them have been broken.  In order to do this, we need to add some code to the Backbone.Validate method.

{% codeblock backbone_validator.js %}
Backbone.Validate = function(model, changedAttributes) {
  
  return (function() {
    this.errors = {};
    this.attributes = _.clone(model.attributes);
    _.extend(this.attributes, changedAttributes);
    _.each(model.validates, function(value, rule) {
      this.validators[rule](value);
    });

    this.validators = {
      required: function(fields) {
       _.each(fields, function(field) {
          if(_.isEmpty(this.attributes[field]) === true) {
            this.addError(field, I18n.t('errors.form.required'));
          }
        });
      }
    };

    this.addError = function(field, message) {
      if (_.isUndefined(this.errors[field])) {
        this.errors[field] = [];
      }
      this.errors[field].push(message);
    };

    return this.errors;
  })();
};
{% endcodeblock %}


That code block can be quite a lot to take in, but I'll try to explain it.  The first thing to not is that it returns the results of a self-invoked function.  The reason for this is to keep the scope of this throughout the Validate method.

Inside this function, we maintain our own references to errors and attributes.  This is because we don't want to do anything that could affect the values that are on the model, which is not the goal, we want to let Backbone deal with that.  We use the clone method of underscore to keep our own reference.

Using the extend method of Underscore, the changedAttributes are merged into our attributes reference, this means that `this.attributes` contains all the attributes present **before** validate was called, and the new ones too.

The code on line 7 is used to call a methods defined in the `this.validators` hash.  Currently there is only one method defined there, but we will expand that in the future and add more validators, such as numericality or format.

As required is the easiest method, that is the first one to implement,  Simply using the `_.isEmpty()` method, you can determine if a required field is present or not.  if it is not present, there is a small helper method to push the error onto the errors object.

If you re-run the test(s) now then you will see that our single test is passing.

##Wrapping up

There are two more tests that we should add before we are done with the required method.  Both should go inside the describe("required") block.

{% codeblock validator_spec.js %}
    it("should allow required on more than one field", function() {
      this.model.validates = {
        required: ["name", "description"]
      };

      this.model.validate();
      expect(this.model.errors).toEqual({"name": ["is required"], "description": ["is required"]});
    });

    it("should not return anything if there are no required fields", function() {
      expect(this.model.validate()).toEqual(undefined);
    });
{% endcodeblock %}


These two tests should be fairly obvious in they're purpose.  One thing of not is in the test with no required fields, we are testing that the validate method returns undefined.  As mentioned earlier, this is because Backbone required you to not return anything if you want to continue with the set/save after validating.

That is it for part 1.  We will take this concept further in part 2.
