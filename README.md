# Using Fast JSON API

## Learning Goals

- Introduce the Fast JSON API gem
- Create serializers using the Fast JSON API gem
- Render related object attributes into JSON data

## Introduction

We've seen that it is entirely possible to create our own service class
serializers from scratch. This issue is common enough, though, that there are
some popular standardized serializer options available for us to use. In this
lesson, we are going to look at one popular option, the [Fast JSON API][fast_jsonapi]
gem and use it to create a close approximation to our JSON data from the
previous lessons.

The files in this lesson were populated using the API-only Rails build. Run
`rails db:migrate` and `rails db:seed` to follow along.

## Introduce the Fast JSON API

The Fast JSON API is a JSON serializer for Rails APIs. It provides a way for us
to generate _serializer_ classes for each resource object in our API that is
involved in customized JSON rendering. We can use these serializer classes to
define the specific attributes we want objects to share or not share, along with
things like related object attributes.

The result is that in our controller actions, rather than writing a custom
`render` each time, we write out a serializer for each object once and use Fast
JSON API to control the way our data is structured.

## Initial Configuration

Before we can see the solution Fast JSON API provides, let's look back at the
problem. We will start at the same place we started when creating our own
service class serializer. This code-along has three resources set up: birds,
locations and sightings:

```rb
class Bird < ApplicationRecord
  has_many :sightings
  has_many :locations, through: :sightings
end
```

```rb
class Location < ApplicationRecord
  has_many :sightings
  has_many :birds, through: :sightings
end
```

```rb
class Sighting < ApplicationRecord
  belongs_to :bird
  belongs_to :location
end
```

We also have one customized controller action:

```rb
class SightingsController < ApplicationController
  def show
    sighting = Sighting.find_by(id: params[:id])
    render json: sighting.to_json(:include => {:bird => {:only =>[:name, :species]}, :location => {:only =>[:latitude, :longitude]}}, :except => [:updated_at])
  end
end
```

This produces a specific set of data, with some but not all related attributes
included:

```js
{
  "id": 2,
  "bird_id": 2,
  "location_id": 2,
  "created_at": "2019-05-14T11:20:37.228Z",
  "bird": {
    "name": "Grackle",
    "species": "Quiscalus Quiscula"
  },
  "location": {
    "latitude": 30.26715,
    "longitude": -97.74306
  }
}
```

With just three objects and some minor customization, rendering has become
complicated. With Fast JSON API, we can extract and separate this work into
Serializer classes, keeping our controller cleaner.

## Setting up Fast JSON API

To include Fast JSON API, add `gem 'fast_jsonapi'` to your Rails project's Gemfile
and run `bundle install`.

Once installed, you will gain access to a new generator, `serializer`.

## Implementing the Fast JSON API

With the new `serializer` generator, we can create serializer classes for all
three of our models, which will be available to us in any controller actions
later.

```sh
rails g serializer Bird
rails g serializer Location
rails g serializer Sighting
```

Running the above generators will create a `serializers` folder within `/app`,
and inside, `bird_serializer.rb`, `location_serializer.rb`, and
`sighting_serializer.rb` have been created. With these serializers, we can start
to define information about each model and their _related_ models we want 
to share in our API.

## Updating the Controller Action

To start using the new serializers, we can update our `render json:` statement
so that it initializes the the newly created `SightingSerializer`, passing in a variable,
just as we did when creating our own service class. The `SightingSerializer` comes with
a method, `serializable_hash`, we will call immediately after initialization:

```rb
class SightingsController < ApplicationController
  def show
    sighting = Sighting.find_by(id: params[:id])
    render json: SightingSerializer.new(sighting).serializable_hash
  end
end
```

This statement can now be used on _all_ `SightingController` actions we want to 
serialize, so if we were to add an `index`, for instance, we just pass in the
array of all sightings as well:

```rb
def index
  sightings = Sighting.all
  render json: SightingSerializer.new(sightings).serializable_hash
end
```

But there is a problem still! If we fire up our Rails server and visit
`http://localhost:3000/sightings/2`, all we see is the following:

```js
{
  "id": "2",
  "type": "sighting"
}
```

The serializer is working, but it behaves a little differently than we're 
used to.

## Adding Attributes

When rendering JSON, controllers will render all attributes available by
default. These serializers work the other way around - we must always specify
what attributes we _want_ to include. In our example, birds have `name` and
`species` attributes and locations have `latitude` and `longitude` attributes,
so to include these we would update both serializers. For sightings, we could
include the `created_at` attribute:

```rb
class BirdSerializer
  include FastJsonapi::ObjectSerializer
  attributes :name, :species
end
```

```rb
class LocationSerializer
  include FastJsonapi::ObjectSerializer
  attributes :latitude, :longitude
end
```

```rb
class SightingSerializer
  include FastJsonapi::ObjectSerializer
  attributes :created_at
end
```

If we go back and check `http://localhost:3000/sightings/2` again, this time,
we will see that the `created_at` attribute is present:

```js
{
  "id": "2",
  "type": "sighting",
  "attributes": {
    "created_at": "2019-05-14T16:39:37.011Z"
  }
}
```

We can also use attributes to access related objects, adding them alongside
normal object attributes:

```rb
class SightingSerializer
  include FastJsonapi::ObjectSerializer
  attributes :created_at, :bird, :location
end
```

This results in our rendered JSON including an `"attributes"` object with 
`"created_at"`, `"bird"`, and `"location"`:

```js
{
  "id": "2",
  "type": "sighting",
  "attributes": {
    "created_at": "2019-05-14T16:39:37.011Z",
    "bird": {
      "id": 2,
      "name": "Grackle",
      "species": "Quiscalus Quiscula",
      "created_at": "2019-05-14T16:39:36.917Z",
      "updated_at": "2019-05-14T16:39:36.917Z"
    },
    "location": {
      "id": 2,
      "latitude": 30.26715,
      "longitude": -97.74306,
      "created_at": "2019-05-14T16:39:36.942Z",
      "updated_at": "2019-05-14T16:39:36.942Z"
    }
  }
}
```

However, here, we have no control over what attributes are included in the
related objects, and so we get _all_ the attributes of `"bird"` and
`"location"`.

## Adding Relationships

Object relationships can be included in serializers in two steps. The first step
is that we include the relationships we want to reflect in our serializers. We
can do this in the same way that we include them in the models themselves. A
sighting, for instance, belongs to a bird and a location, so we can update the
serializer to reflect this:

```rb
class SightingSerializer
  include FastJsonapi::ObjectSerializer
  attributes :created_at
  belongs_to :bird
  belongs_to :location
end
```

However, when visiting `http://localhost:3000/sightings/2`, Fast JSON API will 
display a new `"relationships"` object, but will give only provide limited
information, including the id of the related object:

```js
{
  "id": "2",
  "type": "sighting",
  "attributes": {
    "created_at": "2019-05-14T16:39:37.011Z"
  },
  "relationships": {
    "bird": {
      "data": {
        "id": "2",
        "type": "bird"
      }
    },
    "location": {
      "data": {
        "id": "2",
        "type": "location"
      }
    }
  }
}
```

Setting these relationships up is necessary for the second step. Now that we
have included relationships connecting the `SightingSerializer` to `:bird` and
`:location`, to include attributes from those objects, the recommended method is
to pass in a second _options_ parameter to the serializer indicating that we want to
_include_ those objects:

```rb
def show
  sighting = Sighting.find_by(id: params[:id])
  options = {
    include: [:bird, :location]
  }
  render json: SightingSerializer.new(sightings, options).serializable_hash
end
```

The result:

```js
{
  "data": {
    "id": "2",
    "type": "sighting",
    "attributes": {
      "created_at": "2019-05-14T16:39:37.011Z"
    },
    "relationships": {
      "bird": {
        "data": {
          "id": "2",
          "type": "bird"
        }
      },
      "location": {
        "data": {
          "id": "2",
          "type": "location"
        }
      }
    }
  },
  "included": [{
      "id": "2",
      "type": "bird",
      "attributes": {
        "name": "Grackle",
        "species": "Quiscalus Quiscula"
      }
    },
    {
      "id": "2",
      "type": "location",
      "attributes": {
        "latitude": 30.26715,
        "longitude": -97.74306
      }
    }
  ]
}
```

Because we have a `BirdSerializer` and a `LocationSerializer`, when including
`:bird` and `:location`, Fast JSON API will automatically serialize their
attributes as well.

## Not Quite the Data Structure We Started With

At the beginning of this lesson, we had the following JSON, but with a messy
controller:

```js
{
  "id": 2,
  "bird_id": 2,
  "location_id": 2,
  "created_at": "2019-05-14T11:20:37.228Z",
  "bird": {
    "name": "Grackle",
    "species": "Quiscalus Quiscula"
  },
  "location": {
    "latitude": 30.26715,
    "longitude": -97.74306
  }
}
```

If you recall from when we created our own service class, since we just moved
the `to_json` call from the controller action to the our serializer, the
JSON data structure looked the same.

Using Fast JSON API, with the use of relationships and passing a second
parameter, we are able to get the same _data_, but in a much different
structure. Fast JSON API is meant to be flexible and easy to implement, and it
definitely is! From this point in the example bird watching application, we
could fill out bird and location controllers and when we add in controller
actions, we already have serializers created and ready to use!

In using Fast JSON API though, we lose the ability to design the structure of
our JSON data.

## Conclusion

There is a lot more you can do with the Fast JSON API gem, and it is worth
reading through their [documentation][contents] to become more familiar with
it. It is possible, for instance, to create entirely custom attributes!

What we covered is enough to get us close to where we were creating our
own customized serializers. We do not get to choose exactly how data gets
serialized the way we do when write our own serializer classes, but we
gain a lot of flexibility by using the Fast JSON API.

The Fast JSON API gem provides a quick way to generate and customize JSON
serializers with minimal configuration. Its conventions also allow it to work
well even when dealing with a large number of related objects.

Overall, the goal of this section is to get you comfortable enough to get Rails
APIs up and running. With practice, it is possible to build a multi-resource
API, complete with many serialized JSON rendering endpoints _within minutes_.

Being able to quickly spin up an API to practice your `fetch()` skills is an
excellent way to get familiar with asynchronous requests. As you move towards
building larger frontend projects, you'll also quickly need a place to persist
data and handle things like login security. Rails as an API will be a critical
asset in your development through the remainder of this course.

## Resources

- [Fast JSON API][fast_jsonapi]

[fast_jsonapi]: https://github.com/Netflix/fast_jsonapi
[compounded]: https://github.com/Netflix/fast_jsonapi#compound-document
[attributes]: https://github.com/Netflix/fast_jsonapi#attributes
[contents]: https://github.com/Netflix/fast_jsonapi#table-of-contents
