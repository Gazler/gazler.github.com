---
layout: post
title: "Subdomains With Phoenix"
date: 2015-07-18 16:22
comments: true
categories: ["phoenix", "elixir", "plug"]
---

A common requirement for applications is to have a subdomain per customer that users belonging to that customer can visit. Example of this include: [Slack](https://slack.com) (https://elixir.slack.com, https://phoenix.slack.com, etc.) and [ReadMe](https://readme.io).

This blog post will go through how to set up your Phoenix application so that it can be used in the same way.

The source code for this repository is available at [https://github.com/Gazler/phoenix-subdomain-demo](https://github.com/Gazler/phoenix-subdomain-demo) - there is a commit to represent each step in this post.

## Getting Started

The first thing we will need is a new Phoenix application. Since it is focused on subdomains, I am going to call it subdomainer:

    mix phoenix.new subdomainer

Once the app has been created and all the dependencies have been installed, start the app and visit it at [http://localhost:4000](http://localhost:4000).

    mix phoenix.server

Next we need to ensure that it is accessible via a separate domain and subomains, so the following needs to be added to your `/etc/hosts` file:

    127.0.0.1 subdomainer.dev foo.subdomainer.dev bar.subdomainer.dev

With these additions, you should also be able to access the application via: [http://subdomainer.dev:4000](http://subdomainer.dev:4000), [http://foo.subdomainer.dev:4000](http://foo.subdomainer.dev:4000) and [http://bar.subdomainer.dev:4000](http://bar.subdomainer.dev:4000)

Currently these all point to the same page, but we are going to modify it so that it displays information about the particular app that we are trying to visit.

## Determining If A Subdomain Has Been Set

<!-- more -->

With the app as it stands, if a user visits http://subdomainer.dev:4000 then we want to display the default Phoenix page that was generated with the application. However if they visit http://foo.subdomainer.dev:4000 or any subdomain then we want to show a different page. For the moment, we won't worry too much about which subdomain is being viewed, only that there is a subdomain present.

We need to configure the application so that it knows which domain is the root domain. This is because you cannot make any guarantees about the number of subdomains. In this instance we know that `subdomainer.dev` is the root and `foo.subdomainer.dev` is a subdomain - however our root could be `app.subdomainer.dev` and the subdomain could be `foo.app.subdomainer.dev`

Replace the following in `config/config.exs` under the `config :subdomainer, Subdomainer.Endpoint` block that should be at the top of your file:

{% codeblock lang:elixir %}
  url: [host: "localhost"]
{% endcodeblock %}

With:

{% codeblock lang:elixir %}
  url: [host: "subdomainer.dev"]
{% endcodeblock %}

We now need to update our endpoint so it knows if a subdomain has been provided in the URL. Create a file `lib/subdomainer/plugs/subdomain.ex` file with the following:

{% codeblock lang:elixir %}
defmodule Subdomainer.Plug.Subdomain do
  import Plug.Conn

  @doc false
  def init(default), do: default

  @doc false
  def call(conn, router) do
    case get_subdomain(conn.host) do
      subdomain when byte_size(subdomain) > 0 ->
        conn
        |> router.call(router.init({}))
      _ -> conn
    end
  end

  defp get_subdomain(host) do
    root_host = Subdomainer.Endpoint.config(:url)[:host]
    String.replace(host, ~r/.?#{root_host}/, "")
  end
end
{% endcodeblock %}

This code here implements the `call/2` function expected by plug. This second argument we expect is the module that will be used if a subdomain is found. This also needs to be added to `lib/subdomainer/endpoint.ex` so that we can ensure our plug is called before the router. Add the following line **before** the `plug :router, Subdomainer.Router` line:

{% codeblock lang:elixir %}
  plug Subdomainer.Plug.Subdomain, Subdomainer.SubdomainRouter
{% endcodeblock %}

You will need to restart your server and start it again (with `mix phoenix.server`) every time you make a change to a file in the `lib` directory as only changes in the `web` directory do not require a reload.

You can validate that this is working by visiting [http://subdomainer.dev:4000](http://subdomainer.dev:4000) which will still work as before, however if you visit [http://foo.subdomainer.dev:4000](http://foo.subdomainer.dev:4000) then you will see an error:

 > undefined function: Subdomainer.SubdomainRouter.init/1 (module Subdomainer.SubdomainRouter is not available)

This is because this router does not exist yet.

## Adding The Subdomain Router

To fix the error we just need to create the SubdomainRouter at `web/subdomain_router.ex`:

{% codeblock lang:elixir %}
defmodule Subdomainer.SubdomainRouter do
  use Subdomainer.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
  end

  scope "/", Subdomainer do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
  end

  # Other scopes may use custom stacks.
  # scope "/api", Subdomainer do
  #   pipe_through :api
  # end
end
{% endcodeblock %}

This fixes the error and now both [http://subdomainer.dev:4000](http://subdomainer.dev:4000) and [http://foo.subdomainer.dev:4000](http://foo.subdomainer.dev:4000) work as before.

We want the subdomain pages to route somewhere else though - we can simply modify the `scope` block in the new router to point to a different controller by changing it to:

{% codeblock lang:elixir %}
  scope "/", Subdomainer.Subdomain do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
  end
{% endcodeblock %}

This will error until you create the required controller, so create `web/controllers/subdomain/page_controller.ex`:

{% codeblock lang:elixir %}
defmodule Subdomainer.Subdomain.PageController do
  use Subdomainer.Web, :controller

  def index(conn, _params) do
    text(conn, "Subdomain home page")
  end

end
{% endcodeblock %}

You will note that `PageController` has been used as the name both times. This name is not important, it just needs to match the path from the `scope` block in the router.

This will work, but you will probably see the following error in your terminal:

 > (exit) an exception was raised: ** (Plug.Conn.AlreadySentError) the response was already sent

This is simple to fix - we just need to prevent additional plugs from running if a subdomain is found modify `lib/subdomainer/plugs/subdomain.ex` to include a call to [Plug.Conn.halt/1](http://hexdocs.pm/plug/0.14.0/Plug.Conn.html#halt/1):

{% codeblock lang:elixir %}
  def call(conn, router) do
    case get_subdomain(conn.host) do
      subdomain when byte_size(subdomain) > 0 ->
        conn
        |> router.call(router.init({}))
        |> halt
      _ -> conn
    end
  end
{% endcodeblock %}

## Customize response based on subdomain

The last thing to do for this is to customize the response based on the which subdomain has been visited. To do this, we will add it to the `private` storage that exists in a `Plug.Conn` which you can read about [in the Plug docs](http://hexdocs.pm/plug/Plug.Conn.html#put_private/3)

We will do this in the `Subdomainer.Router` module where we did the initial check to see if a subdomain exists by modifying the `call` function:

{% codeblock lang:elixir %}
  def call(conn, opts) do
    case get_subdomain(conn.host) do
      subdomain when byte_size(subdomain) > 0 ->
        conn
        |> put_private(:subdomain, subdomain)
        |> router.call(router.init({}))
        |> halt
      _ -> conn
    end
  end
{% endcodeblock %}

We can then retrieve this value in the index action of our `Subdomainer.Subdomain.PageController` like so:

{% codeblock lang:elixir %}
  def index(conn, _params) do
    text(conn, "Subdomain home page for #{conn.private[:subdomain]}")
  end
{% endcodeblock %}

And that's it! All of the following pages should work and show the correct page [http://subdomainer.dev:4000](http://subdomainer.dev:4000), [http://foo.subdomainer.dev:4000](http://foo.subdomainer.dev:4000) and [http://bar.subdomainer.dev:4000](http://bar.subdomainer.dev:4000)

From here you can extend this to the needs your app. One common task for an application with this feature is to lookup an application from your database based on the subdomain provided and show a 404 if one is not found. Since you know by the time the `Subdomainer.SubdomainRouter` is reached in the request, the subdomain will be set, you could add a `plug` to the `pipeline` to perform this check before the controller action is reached.
