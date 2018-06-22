---
layout: post
title:      "Handling JSON Responses for Deeply Nested Data"
date:       2018-06-22 16:22:08 +0000
permalink:  handling_json_responses_for_deeply_nested_data
---


Doing something for the first time is an important experience. The toil involved can lead to important discoveries and patterns that many can benefit from. But the quality of information is not necessarily the best considering what else may be available. I am not experienced at all, but I did go through some struggles making my Rails server respond with the JSON I wanted. 

I will show you what my goal was, how I thought conventional-sensitive tools should have achieved it, and how I ended up having to navigate the issue to reach the original goal.

A bare Rails server handles the front end by rendering model information together with HTML templates. Rails usually sends this data over from the controller to whoever requested it. If the browser itself requests this information, (no Javascript,) then Rails sends the data to the browser and the browser shows you the page data.

Lets say you have an index of all Users for an application. You click on a user’s name, you send a request to the server, the server decides what it will send you in a controller method, which finds the user in the database. When the Rails server looks up the appropriate HTML file it will send the browser, ERB lets the server read through that whole file and inject code evaluation as needed. That resultant string is what is sent tot he browser. Even if you have `@user.posts.comments.each` in a view file, the server will handle that appropriately.

On my User index page, I decided to render the whole show page of the user right under the link to their show-page, using Javascript. 

Take a look at the User show-page:

![User-Show-Page](https://s8.postimg.cc/73zpc20kl/Screen_Shot_2018-06-22_at_10.42.58.png)

a User has\_many days, (only 7,) and each day has\_many tasks. In ERB, this is no issue to iterate over. (Although the design is probably not ideal what-so-ever.) This is done like so:

```erb
<% @user.days.each do |day| %>

  day.name
  
  <% day.accomplishments.each do |acc| %>
    
    acc.title
    
  <% end %>
 
<% end %>
```

Now, we want this whole block of stuff popping out of a user’s name on the index page. That means on the index page, we are going to attach an event listener to the link we WERE using, and send a request to users\#show part of the server using Javascript instead of the browser. That means the server will respond to our Javascript instead of our browser. 

We may have thought to keep all of our ERB running the same way and just send that huge HTML string back to Javascript and render it right into the page using `document.write()` stuff. But that isn’t how this project works! We need JSON from the controller. A JSON is a lot smaller than that huge string and a lot easier to parse and rewrite the DOM with. 

Enter serialization. We need to make the controller take `@user.days.accomplishments` and send that as a JSON.

This is supposed to be easy with the `active\_model\_serializers` gem. The controller can render a JSON constructed out of an object all on it’s own, but to fine tune the JSON and make it adhere to various standards, and to include it’s nested relationships, we use this gem. Ok.

remember that we configured the serializer to give us ‘official’ JSON objects:

```ruby
ActiveModelSerializers.config.tap do |c|
  c.adapter = :json_api
  c.jsonapi_include_toplevel_object = true
  c.jsonapi_version = "1.0"
end
```

The serializers:

```ruby
class UserSerializer < ActiveModel::Serializer
  attributes :id, :first_name, :last_name, :email
  has_many :days
end

class DaySerializer < ActiveModel::Serializer
  attributes :id, :name
  has_many :accomplishments
end

class AccomplishmentSerializer < ActiveModel::Serializer
  attributes :id, :user_id, :title, :effect
end
```

When the controller hits:

```ruby
format.json { render json: @user }
```

You would think it would serialize all the nested relationships as we defined them in the serializer. That totally doesn’t work. Check it out:

```json
{
  "data": {
    "id": "1",
    "type": "users",
    "attributes": {
      "first-name": "Yehuda",
      "last-name": "Makarov",
      "email": "yehudamakarov@gmail.com"
    },
    "relationships": {
      "days": {
        "data": [
          {
            "id": "1",
            "type": "days"
          },
          {
            "id": "2",
            "type": "days"
          },
          {
            "id": "3",
            "type": "days"
          },
          {
            "id": "4",
            "type": "days"
          },
          {
            "id": "5",
            "type": "days"
          },
          {
            "id": "6",
            "type": "days"
          },
          {
            "id": "7",
            "type": "days"
          }
        ]
      }
    }
  },
  "jsonapi": {
    "version": "1.0"
  }
}
```

where are the attributes I defined for days? What about my accomplishments? Let’s try what is referenced and suggested at [https://github.com/rails-api/active\_model\_serializers/issues/2011](https://github.com/rails-api/active_model_serializers/issues/2011) and in the docs at [https://github.com/rails-api/active\_model\_serializers/blob/a032201a91cbca407211bca0392ba881eef1f7ba/docs/general/adapters.md\#included](https://github.com/rails-api/active_model_serializers/blob/a032201a91cbca407211bca0392ba881eef1f7ba/docs/general/adapters.md#included)

```ruby
render json: @user, include: ['days', 'days.accomplishments']
```

Check out the new JSON:

```json
{
  "data": {
    "id": "1",
    "type": "users",
    "attributes": {
      "first-name": "Yehuda",
      "last-name": "Makarov",
      "email": "yehudamakarov@gmail.com"
    },
    "relationships": {
      "days": {
        "data": [
          {
            "id": "1",
            "type": "days"
          },
          {
            "id": "2",
            "type": "days"
          },
          {
            "id": "3",
            "type": "days"
          },
          {
            "id": "4",
            "type": "days"
          },
          {
            "id": "5",
            "type": "days"
          },
          {
            "id": "6",
            "type": "days"
          },
          {
            "id": "7",
            "type": "days"
          }
        ]
      }
    }
  },
  "included": [
    {
      "id": "1",
      "type": "days",
      "attributes": {
        "name": "Sunday"
      },
      "relationships": {
        "accomplishments": {
          "data": [
            {
              "id": "5",
              "type": "accomplishments"
            },
            {
              "id": "6",
              "type": "accomplishments"
            },
            {
              "id": "8",
              "type": "accomplishments"
            },
            {
              "id": "9",
              "type": "accomplishments"
            },
            {
              "id": "11",
              "type": "accomplishments"
            },
            {
              "id": "12",
              "type": "accomplishments"
            },
            {
              "id": "13",
              "type": "accomplishments"
            },
            {
              "id": "14",
              "type": "accomplishments"
            },
            {
              "id": "15",
              "type": "accomplishments"
            },
            {
              "id": "16",
              "type": "accomplishments"
            }
          ]
        }
      }
    },
    {
      "id": "5",
      "type": "accomplishments",
      "attributes": {
        "user-id": 1,
        "title": "123",
        "effect": "123",
        "human-time": " 7:00 PM"
      }
    },
    {
      "id": "6",
      "type": "accomplishments",
      "attributes": {
        "user-id": 1,
        "title": "234",
        "effect": "javascript",
        "human-time": " 7:00 PM"
      }
    },
    {
      "id": "8",
      "type": "accomplishments",
      "attributes": {
        "user-id": 1,
        "title": "345",
        "effect": "345",
        "human-time": " 7:00 PM"
      }
    }
  ],
  "jsonapi": {
    "version": "1.0"
  }
}
```

Now, if you look at this JSON carefully you will see that we didn’t solve much. The included array has indices that are both days and the day’s accomplishments. Wherever the accomplishments are nested properly in the day, the accomplishment object only has an ID. 

This is actually the doing of the configuration we passed to `active\_model\_serializers`. If you turn off all the configuration the JSON object response will be perfectly nested exactly how you would want.

Great so if I want to be compliant I have to use awful JSON?

We can also try adding this to the configuration:

```ruby
ActiveModelSerializers.config.default_includes = "**"
```

As per the docs, \* or \*\* are wildcards we can tell the serializer to use. The serializer should then include either 1 level of nested data or all levels of nested data.

```ruby
ActiveModelSerializers.config.tap do |c|
  c.adapter = :json_api
  c.jsonapi_include_toplevel_object = true
  c.jsonapi_version = "1.0"
  c.default_includes = "**"
end
```

But the `jsonapi\_version = “1.0”` still has everything in a big `included` array and its not properly nested.

Solution:

Including the days which are one level deep in the resource seems to work. We have the data of the days, and the days themselves nest their accomplishments properly in their own JSON. So our work will start in the day serializer. Let’s switch the controller back to only including days. Because things in the included array don’t get nested more than one level deep.

```ruby
format.json { render json: @user, include: ['days'] }
```

Now lets look at our DaySerializer:

```ruby
class DaySerializer < ActiveModel::Serializer
  attributes :id, :name
  has_many :accomplishments
end
```

The has\_many is putting accomplishments in the “relationships” of days, but we don’t get any of the attributes of an accomplishment besides `id`. Lets not use has\_many then. We will have to make a custom method:

```ruby
class DaySerializer < ActiveModel::Serializer
  attributes :id, :name, :accomplishments

  def accomplishments
    object.accomplishments.map do |acc|
      # make an object using the accomplishment attributes
      # :id, :user_id, :title, :effect, :human_time
      {
        id: acc.id,
        user_id: acc.user_id,
        title: acc.title,
        # etc
      }
    end
  end
end
```

Now, when we `include days` each day will have an array of accomplishments in its attributes:

```json
{
  "data": {
    "id": "1",
    "type": "users",
    "attributes": {
      "first-name": "Yehuda",
      "last-name": "Makarov",
      "email": "yehudamakarov@gmail.com"
    },
    "relationships": {
      "days": {
        "data": [
          {
            "id": "1",
            "type": "days"
          },
          {
            "id": "2",
            "type": "days"
          },
          {
            "id": "3",
            "type": "days"
          },
          {
            "id": "4",
            "type": "days"
          },
          {
            "id": "5",
            "type": "days"
          },
          {
            "id": "6",
            "type": "days"
          },
          {
            "id": "7",
            "type": "days"
          }
        ]
      }
    }
  },
  "included": [
    {
      "id": "1",
      "type": "days",
      "attributes": {
        "name": "Sunday",
        "accomplishments": [
          {
            "id": 5,
            "user-id": 1,
            "title": "123"
          },
          {
            "id": 6,
            "user-id": 1,
            "title": "234"
          },
          {
            "id": 8,
            "user-id": 1,
            "title": "345"
          },
          {
            "id": 9,
            "user-id": 1,
            "title": "789"
          },
          {
            "id": 11,
            "user-id": 1,
            "title": "only giving a day of week"
          },
          {
            "id": 12,
            "user-id": 1,
            "title": "sdfgfdgjh"
          },
          {
            "id": 13,
            "user-id": 1,
            "title": "xcvxcv"
          },
          {
            "id": 14,
            "user-id": 1,
            "title": "updated"
          },
          {
            "id": 15,
            "user-id": 1,
            "title": "asd"
          },
          {
            "id": 16,
            "user-id": 1,
            "title": "sdfhsdfhasdfh"
          }
        ]
      }
    }
  ],
  "jsonapi": {
    "version": "1.0"
  }
}
```

There may be better ways to do this. I do not know enough about the JSON API specification and its importance. It could very well be designed EXACTLY so we don’t do something like this.

For instance we could first make a response to a user’s days, and render links to all those days using those ids we saw in the first responses we were getting before all the monkey patching. Then we could push on a certain day to see its accomplishments and links to each one. And then push an accomplishment to see its details. We would send a new request for each link and this way we don’t bomb the server with huge serialization tasks and the browser with these big rewrites.

These are great questions to be asking and things to be thinking about to manage a Rails API properly. Then there is eager loading the models in the controller.

Before that nested serialization job I want to make sure I already made the call to the database and I have those objects in hand before the parse.

```ruby
@user = User.includes(days: :accomplishments).find(params[:id])
```

In the terminal you will see something like this after a serialization job:

```sh
  ↳ app/controllers/application_controller.rb:9
[active_model_serializers] Rendered UserSerializer with ActiveModelSerializers::Adapter::JsonApi (12.42ms)
Completed 200 OK in 81ms (Views: 29.4ms | ActiveRecord: 3.2ms)
```

For one of the other examples earlier it was:

```sh
  ↳ app/controllers/application_controller.rb:9
[active_model_serializers] Rendered UserSerializer with ActiveModelSerializers::Adapter::JsonApi (31.02ms)
Completed 200 OK in 96ms (Views: 45.1ms | ActiveRecord: 3.6ms)
```

Try to see how long certain things take for the API. Trying to NOT have absurdly nested resources and NOT have to serialize these into super JSONs is key in structuring the data relationships and the tables.

This is also why User Interface and User Experience must be a primary factor in managing data structure. Which is part of the importance of a Full Stack mindset. (Not necesarilly hard skill set.)

I’m around if you need me :)














