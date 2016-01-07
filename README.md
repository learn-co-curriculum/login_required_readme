# Requiring login

## Overview

Sometimes you want to require that a user is logged in to access a route. Here's how.

## Objectives
  1. Restrict a route to logged in users.
  2. Skip a filter for particular controller actions.

## First pass: manual checks

Let's say we have a `DocumentsController`. Its `show` method looks like this:

   def show
     @document = Document.find(params[:id])
   end

Now let's add a new requirement: documents should only be shown to users when they're logged in. Logged in users have `:user_id` set on their `session`.

The first thing you might do is to just add some code into `DocumentsController#show`:

  def show
    return head(:forbidden) unless session.include? :user_id
    @document = Document.find(params[:id])
  end

We're using the phrase `unless session.include?` rather than `unless session[:user_id]`, because a user_id of 0 is still a valid user id, but would evaluate falsey, and reject the request.

## Refactor

This code works fine, so you use it in a few places. Now your DocumentsController looks like this:

  class DocumentsController < ApplicationController
    def show
      return head(:forbidden) unless session.include? :user_id
      @document = Document.find(params[:id])
    end

    def index
      return head(:forbidden) unless session.include? :user_id
    end

    def create
      return head(:forbidden) unless session.include? :user_id
      @document = Document.create(author_id: user_id)
    end

    def update
      return head(:forbidden) unless session.include? :user_id
      @document = Document.find(params[:id])
      # code to update a document
    end    
  end

That doesn't look so DRY. I really wish there were a way to ask Rails to run a check before any controller action.

Fortunately, Rails gives us a solution: [`before_action`][filters]. We can refactor our code like so:

  class DocumentsController < ApplicationController
    before_action :require_login
    
    def show
      @document = Document.find(params[:id])
    end

    def index
    end

    def create
      @document = Document.create(author_id: user_id)
    end

    private

    def require_login
      return head(:forbidden) unless session.include? :user_id    
    end
  end

Let's look at the code we've added:

    before_action :require_login

This is a call to the ActionController class method `before_action`. `before_action` registers a [filter]. A filter is a method which runs before, after, or around, a controller's action. In this case, the filter runs before all DocumentsController actions, and kicks requests out with `403 Forbidden` unless they're logged in.

## Skipping filters for certain actions

What if we wanted to let anyone see a list of documents, but keep the `before_action` filter for other `DocumentsController` methods? We could do this:

  class DocumentsController < ApplicationController
    before_action :require_login
    skip_before_action :require_login, only: [:index]

    # ...
  end

This class method,

    skip_before_action :require_login, only: [:index]

Tells Rails to skip the `require_login` filter only on the `index` action.

## Resources
  * [Action Controller Overview — 8, Filters][filters]

[filters]: http://guides.rubyonrails.org/action_controller_overview.html#filters
