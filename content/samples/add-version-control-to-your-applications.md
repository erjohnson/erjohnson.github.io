---
title: "Add Simple Version Control to Your Applications"
date: "2022-02-02"
summary: "Blog post for Orchestrate.io"
---

These days users switch between multiple devices, collaborate remotely with co-workers and even participate in crowd curated content. From a database perspective, there are plenty of ways to potentially overwrite data. Though we've built some features to avoid duplicate writes into Orchestrate to avoid duplicate writes, there are still applications that benefit from looking through previous versions. For example, a text editor where you can revert to previous revisions.

In this post, I'll walk you through building a simple text editor application in Ruby using [Refs](##) to power undo functionality. Refs are hashes of the content in an item stored within Orchestrate. So if two "versions" of an item are identical, they'll have the same Ref. For our use case, that's just fine.

For this tutorial, I chose [Sinatra](##) to build out the application. The source code for this app is [available on GitHub](##).

## Getting Started

Ensure you have [Ruby](##) and [Bundler](##) installed.

### Directory Structure

Let's start by setting up the structure for the app:

```text
/my_app
	/public
		/css
	/views
```

### Dependencies

- [Sinatra](##): DSL for creating web apps in Ruby
- [Sinatra::Contrib](##): common Sinatra extensions (we'll be using the reloader extension)
- [Orchestrate Gem](##): Ruby library to interface with the Orchestrate API
- [Thin](##): fast & simple Ruby web server
- [Dotenv](##): Loads environment variables from `.env` files

From your terminal, navigate to your app's root folder and run `bundler init` to create a new Gemfile. Then add the gems! It should look something like this:

```ruby
# Gemfile

source "https://rubygems.org"

gem "sinatra", git: "https://github.com/sinatra/sinatra.git"
gem "sinatra-contrib", git: "https://github.com/sinatra/sinatra-contrib.git"
gem "orchestrate"
gem "thin"
gem "dotenv"
```

*Due to issues with the version of Sinatra on [RubyGems](##) and the most recent version of [Rack](##), I recommended installing Sinatra & Sinatra-Contrib from the GitHub source.*

Install the gem dependecies by running `bunder install` in your terminal.

## Storing the API Key with Dotenv

Next we'll grab the Orchestrate API key from our [application's Dashboard](##). Check out the [Getting Started](##) guide for step-by-step instructions on creating an Orchestrate application and obtaining its API key.

To keep our Orchestrate API key a secret, we'll use [Dotenv](##). Dotenv lets us store environment variables in a `.env` file during development and reference the variables in our code.

Go ahead and create the `.env` file in your app's root folder. In it we'll add our API key:

```text
ORC_API_KEY=MY_SECRET_KEY
```

If you're using git you can add this line to your `.gitignore` file:

```text
# Ignore dotenv
/.env
```

This tells Git to ignore the file and leave it out of commits (keeping our API key secret!). GitHub has a good introduction to [ignoring files](##) if you'd like to learn more.

## Creating the Initial App

Let's add two more files into the app's root directory: `app.rb` and `config.ru`.

`app.rb` is where the main logic of our application will be. Open up `app.rb` in your favorite text-editor and we'll get it started by adding the following:

```ruby
require 'bundler'
require 'bundler/setup'
require 'sinatra/reloader'

# Have Bundler require default Gems
Bundler.require

Dotenv.load

get '/' do
  "Hello World!"
end
```

Next open config.ru and add:

```ruby
require "./app"
run Sinatra::Application
Let's check if everything is working so far.
```

Run `rackup` on the command-line:

```text
// Sample output
$ rackup
Thin web server (v1.6.3 codename Protein Powder)
Maximum connections set to 1024
Listening on localhost:9292, CTRL+C to stop
Visit localhost:9292 in the browser should return "Hello World!"
```

At this point, your app directory should look something like this:

```text
/my_app
	/public
		/css
	/views
	.env
	.gitignore
	app.rb
	config.ru
	Gemfile
	Gemfile.lock
```

## Adding Styles

Before we go further, let's add some stylesheets. I chose [Skeleton](##), a fairly straight forward CSS boilerplate. Put your styles under `public/css`.

## Connect to Orchestrate

There are a few ways to do this. In Sinatra we can set up Helpers; methods which both routes and templates can use. Let's define a helper method in `app.rb` to connect to Orchestrate:

```ruby
require 'bundler'
require 'bundler/setup'
require 'sinatra/reloader'
Bundler.require
Dotenv.load

helpers do
  def client
    @client ||= Orchestrate::Client.new(ENV['ORC_API_KEY'])
  end
end
```

Now when we use client in our routes, an `Orchestrate::Client` object will be returned.

## Defining Routes

Our app would be fairly dull if all it did was print out "Hello World", so let's remedy that by adding some more routes:

```ruby
# root
get '/' do

  # Retrieve a list of items from the 'documents' collection
  @list = client.list(:documents).results

  # If the list is empty, render the 'no_docs' view
  if @list.empty?
    erb :no_docs
  else
    @list
    erb :index
  end
end

# new document
get '/new' do
  erb :new
end

# create document
post '/new' do

  # Lowercase title, strip whitespace to use as Key
  @doc_id = params[:title].downcase.chomp.gsub(' ', '-')

  @new_doc = { title: params[:title], content: '' }

  @res = client.put(:documents, @doc_id, @new_doc)

  if @res.success?
    redirect to("/document/#{@doc_id}")
  end
end

# view document
get '/document/:id' do

  @res = client.get(:documents, params[:id])

  if @res.success?
    @title = @res.body['title']
    @content = @res.body['content']
    @id = params[:id]
  end

  erb :show
end
```

## Add Views

Next we need to create some templates to render the views:

- `layout.erb`: Default layout template, renders views.
- `index.erb`: Home view.
- `new.erb`: New document view.
- `show.erb`: View & edit document view.
- `no_docs.erb`: "Not found" view.

Put these files in the views folder in your application directory. I made a Gist containing the views which can be found [here](##).

## Save & Undo

For saving and undoing changes we can add new routes. Here's how mine look:

```ruby
put '/document/:id' do

  @doc = { title: params[:title], content: params[:content] }

  @res = client.put(:documents, params[:id], @doc)

  status 201 if @res.success?
end

get '/undo/:id' do

  # Tell Orchestrate to limit results to 1
  # and skip the 1st result (current document value)
  options = { limit: 1, offset: 1, values: true }

  @prev_ref = client.list_refs(:documents, params[:id], options)

  @prev_ref.on_complete do
    @prev_val = @prev_ref.results.first['value']
  end

  @res = client.put(:documents, params[:id], @prev_val)

  if @res.success?
    redirect to("/document/#{params[:id]}")
  end
end
```

Instead of a "Save" button in the view, I added a script to `show.erb` that saves the document when the user stops typing:

```javascript
$(document).ready(function(){
	$("textarea#editor").val("<%= @content %>");
	function save() {
	  var title = $("h1#title").text();
	  var content = $("textarea#editor").val();
	  $.ajax({
	    url: '/document/<%= @id %>',
	    type: 'PUT',
	    data: { title: title, content: content}
	  });

	  $('#saved').show();

	  setTimeout(function() {
	    $('#saved').hide();
	  }, 4000);
	};

	var timeoutId;

	$('#editor').on('input propertychange', function() {
	  clearTimeout(timeoutId);
	  timeoutId = setTimeout(function() {
	    save();
	  }, 1000);
	});
});
```

To hide the "Saved!" message by default, I added this to my css:

```css
#saved {
  display: none;
}
```

Awesome! Now whenever we update our content, we can undo those changes if we mess up. You can even undo the changes that someone else makes.

If you want to skip the step by step, you can [fork the repo on GitHub](##) and start playing with the complete app locally. A great next step for this project would be to view a list of versions in the order created, allowing users to choose which one to view and revert.

Or, better yet, use this quick text editing app as a sample to better leverage [Refs](##) in your next or existing applications.
