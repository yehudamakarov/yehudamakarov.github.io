---
layout: post
title:      "A Mark Down Blog"
date:       2018-08-14 03:39:25 +0000
permalink:  a_mark_down_blog
---


### General Points

* You’re starting a rails/react project from scratch, maybe your first.

1. Front End
  1. Pick your UI framework.
    1. Be patient with it, everyone needs to know CSS.
  2. Start imagining how the interface will interact with the database as infrequently as possible. This will often define a large part of the application’s scalability.
2. Back End
  1. Serialize the data responsibly.
    1. Eager loading.
    2. Minimize nested relationships.
3. Try to come out with a front end that isn’t totally dependent on the backend for most components being mounted/rendered.

### Before Writing Anything

Beginning the project, there was a simple goal in mind: Have a blog, where there could be an admin view, containing a whole preview of the blog as viewable and editable.

We would use Rails for a a database and API endpoints, and the admin user account. React and Redux would handle asking the database for all the info and also provide viewing components.

Right away, some design issues had to be straightened out.

* How would we secure the endpoints of creating and editing posts? How would the admin be logged in or logged out?
* How would the routes be set up so that any passers-by could go to `’/‘`​ and see the same component that we would see in the edit view of the `​‘/admin’`​ route?
* How would we filter by tags?
* How would we make sure the redux store is always in sync wwith the database?

It’s really important to realize that the first time trying anything, our attempts can’t possibly be the most polished. So I framed my outlook with this idea and focused on making a working form that would submit a record first.

Fashion a working piece of UI that will need the backend in some way. Build that first. This could be:

* A form for a record
* A signup for Users
* A component for a dashboard
* Or something similar

### The ‘New’ Form

The form for making a new blog post was going to use an exported markdown file from Quiver, an awesome note-taking app. Let’s imagine:

You’re working on something in your code and you get a little curious about something, or say, something isn’t working like you thought it should. You start googling around, packing fact after fact into your little napsack. And then maybe you hit that perfect stack-overflow question. Awesome!

A lot of people like to encapsulate that little escapade and make a blog post out of it. At the very least, you should take some notes about whatever you just picked up. Because that might come up again, and it’s good to be prepared. Either way, writing it out helps it be remembered and understood.

I put these things in my quiver notebook.

Now, wouldn’t it be nice if those notes could easily become blog posts? I wanted my ‘new’ form to handle this. (Quiver exports a markdown file. With all the images, code snippets, etc. that went into that note. )

* It needed to take my exported markdown file from my Quiver and have it database-ready.
* It needed to handle tag input.
* It needed to automatically upload any of the images in the exported markdown, send them to imgur and progrmatically insert those new URL’s into the markdown content for me.
* It needed to implement a similar feature for a cover image (which might honestly be a sketch or graph or something I made to help myself understand whatever it was that I was struggling with).
* It needed to validate appropriately.

It is important to periodically take a step back into an architectural stand-point. Especially the first time doing something. But on the other hand, the proper architectural vantage point is really only achieved through writing code out and scrutinizing its purpose.

What happened was, after getting caught up in Material UI for the first time, There was a mess of code and files that worked alright. Scalability wasn’t a necessity here as we have only one user. And the different points of uploading, previewing and replacing image files got a little cumbersome.

But this is exactly where the ‘fun' of refactoring begins.

One important thing I realized I should have done differently:

Loading the app with the content retreived from the database, there is basically an array of posts and tags in two respective property names of the store. When submitting a new record, there is a POST request sent out by the app and and a new record is made.

At this point we need to make sure the new record is in the store so the store is in sync with the database.

###  Make the database do as little work as possible.

Somehow I ended up just fetching the database contents again in a callback to the POST action. Meaning the app sends the POST, makes the record in the DB, and immediately afterwards, that same Redux action makes a GET to the DB for all the posts; the new one included. Then the store is refilled with that GET’s response. This is silly design.

Instead, we should simply use the server's response to the POST (that the Rails controller sends out upon creation of a new record being created) to fill the store ONLY with the new value. We should not be fetching all the posts just because we made one. We have a response already just from making the post, we can use it to insert the new record into the store. If the creation of the record fails, we will not add the record to the store.

And in the update functionality we can just copy/paste the same logic. The only time the app should need to contact the DB is when the app loads the first time. The other times we make changes to the DB, we should be getting responses containing any edited content. Those responses can be used to update the store locally. Less calls on the DB, happier app. Server side validation runs before the responses come in, so if we use callbacks in things like `​.then`​ and `​.catch`​ we will only update the store in sync with the DB.

###  Authentication

We only needed an admin user, but if this would be pushed to production, you wouldn’t want someone being able to access your admin dashboard.

I saw a JSON Web Token approach out there. It is pretty nice. But I was used to working with the session that Rails handles. This isn’t default in the API Rails version. I liked it because I just didn’t want to worry about any security in the client code for now.

I pushed my login and authentication logic to Rails for the sole admin user. We would only need to make a login form, a single User row in a Users table, and use BCrypt for password hashing.

Now how do you tell the browser to remember who logged in? The browser would need some way to identify itself to the Rails app. In Rails you might be able to just:

```ruby
class SessionsController < ApplicationController
    def create
        @user = User.first.authenticate(params[:password]) 
        if @user
            render json: @user
            session[:user_id] = @user.id
        else
            render json: { "errors": { "authentication": "failed" } }, status: 403
        end
    end
end
```

And you would need to run one of these before any controller actions requiring authentication:

```ruby
def require_login
    return head(:forbidden) unless logged_in?
end

def logged_in?
    !!current_user
end

def current_user
    @current_user ||= User.find_by(id: session[:user_id])
end
```

But think about this: How does Rails know who is who? It is just a database on a server with open endpoints. You send a POST request to the sessions\_controller and you set this session[:user\_id] to an integer.

Usually there is more than one user communicating with the Rails app at once. Whenever a browser contacts the IP address of the Rails server, it spins up a new instance of the class you deifne with `​class ApplicationController`​. So there can be a lot of these objects which are sort of processes floating around on your Rails app. Each one of these genral connections that the Rails app has with the browser is what you would call a session. And Rails remembers each one distinctly based on a cookie in the browser.

I set this up in my Rails API like so in the config/application.rb file:

```ruby
config.middleware.use ActionDispatch::Cookies
config.middleware.use ActionDispatch::Session::CookieStore, key: (generate a random secret and put the string here), expire_after: 1.hour
```

Now a browser contacting this app will get a cookie that Rails can use to recognize the session. Meaning, If a browser contacts the Rails app and in one of the controller methods, we edit the session data, then the response that we send back to the client will store the cookie in the browser. Check the network tab. It should reflect this cookie being sent in the request headers. If it it isn’t something is not configured right. Now a request that we send to the browser should include that cookie. That is what Rails session uses to figure out which session it is dealing with.

###  Realizing this isn’t the whole picture.

Sometimes you realize you spend a few hours thinking of minor details. Definitely worth it to employ more than this though, because if someone has the admin’s cookie, then what is stopping them from making the server think that their **own** browser is the true owner of the session cookie. Because Rails will get a cookie name and value, that is sent as a response header. (again this is in the developer tools under the developer tools:

![Screen Shot 2018-08-13 at 21.50.28.png](https://i.imgur.com/OdOjNlT.png)

That is a POST request to posts, asking to create a post. We send that huge set of strings as response headers.)

Rails gets that in the routes portion of your app. when the controller instance on the server computing unit uses the session hash, it is referencing these values that the browser sent. There is much more going on and I don’t know very much. But this is essentially your session identity if you want to use the session store. any other session data is accessible with that cookie as far as I know. So if you’re not using SSL, maybe this is problematic.

JWT is now very helpful. And you might want to make sure the session cookies expire after a given amount of time.



























