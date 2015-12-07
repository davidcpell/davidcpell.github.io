---
layout: post
title: Mapping Data from External APIs to ActiveRecord Objects with the Representable Gem - Part III
---

## Mapping JSON with Representable

Parts [I](http://davidpell.net/Mapping-Data-From-External-APIs-to-ActiveRecord-Objects-with-the-Representable-Gem-Part-I/) and [II](http://davidpell.net/Mapping-Data-From-External-APIs-to-ActiveRecord-Objects-with-the-Representable-Gem-Part-II/) of this series covered the basic questions of what an API is and how our browsers and command prompts can make "requests" to API "endpoints" to get "responses" in the form of text documents (HTML and JSON formats). In this post I'm going to cover how the Representable gem simplifies the process of mapping JSON responses to ActiveRecord objects so we can save them quickly and easily to our database. 

 Before diving in, it might help to start with a concrete example of some of the problems we might run into when trying to build ActiveRecord objects from JSON responses. Going back to our supervillain application from the previous posts, let's say we have a `Supervillain` model that looks like this:

```ruby
class Supervillain < ActiveRecord::Base
  has_many :nicknames
end
```

Our `Supervillain` table has the following columns as specified in `schema.rb`:

```ruby
  create_table "supervillains", force: :cascade do |t|
    t.string   "name"
    t.string   "publisher"
    t.integer  "first_year"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
  end
```

Now let's imagine that when we make a request to an API that returns Supervillain JSON objects we get the following response:

```json
{ 
  "villain_id": 123,
  "name": "Lex Luther",
  "publisher": {
    "id": 1,
    "name": "DC Comics",
    },
  "nicknames": [
      { "id": 1, "name": "Mockingbird" },
      { "id": 2, "name": "Kryptonite Man" }
    ],
  "year_introduced": 1940
}
```

There are a number of hurdles we'll need to jump in order to make this kind of response work smoothly with our model. If we try to initialize our own Supervillain object with this JSON response directly, look what happens:

```ruby
# the above JSON response has been cached as the variable `json`

irb(main):034:0> Supervillain.new(json)
ArgumentError: When assigning attributes, you must pass a hash as an argument.
```

First we see that we can't pass JSON to the `.new` method directly. `.new` will work with a hash, but not a JSON document (string). However, ActiveRecord *does* supply the [instance method](http://apidock.com/rails/ActiveModel/Serializers/JSON/from_json) `#from_json`, which we can call on a blank new object with our JSON as an argument. Yet this results in a different exception being raised:

```ruby
irb(main):035:0> Supervillain.new.from_json(json)
ActiveRecord::UnknownAttributeError: unknown attribute 'villain_id' for Supervillain.
```
When we set values on new instances of our models, ActiveRecord looks for a column in the database schema that matches the attribute. Here we are implicitly trying to set an attribute called `villain_id` because we passed a JSON object with that key to the `#from_json` method. The problem, of course, is that our table doesn't have a `villain_id` column! Why not? We don't really care how the external API internally identifies villains.  

Now that we're stuck, let's install Representable and start looking at how it will make this process a lot easier!

#### Installation

Representable is easy to install. Just add it to your `Gemfile`:

```ruby
gem 'representable'
```

...and run `bundle install` from your command prompt.

#### General Usage

There are a couple different but related ways to use Representable. I'm going to stick with the "module" approach in this tutorial. With the module approach we create "Representer" modules that are named after our models (so in this case we'll make a `SupervillainRepresenter`) and then extend the modules directly onto instances of the model. The modules make use of three main keywords: `property`, `nested`, and `collection` (more on them later). Once we extend the Representer onto our model instance, we can call `from_json` like before, but with much better results! 

Our representers go in their own directory: `app/representers` and when we use them, we'll end up with a line of code looking like this: 

```ruby
villain = Supervillain.new.extend(SupervillainRepresenter)
villain.from_json(json)
```

#### TDD and Attribute Whitelisting (Strong Params)

Making Representers easily lends itself to a Test Driven workflow, even for people who are new to the approach. Here's a simple spec file with one assertion, 

```ruby
# spec/representers/supervillain_representer_spec.rb
require 'rails_helper'

describe SupervillainRepresenter do 
  it 'maps Supervillain attributes correctly' do 
    json = { 
      "villain_id": 123,
      "name": "Lex Luther",
      "publisher": {
        "id": 1,
        "name": "DC Comics",
        },
      "nicknames": [
          { "id": 1, "name": "Mockingbird" },
          { "id": 2, "name": "Kryptonite Man" }
        ],
      "year_introduced": 1940
    }.to_json

    villain = Supervillain.new.extend(SupervillainRepresenter)
    villain.from_json(json)

    expect(villain.name).to eq('Lex Luther')
  end
end
```

Let's run the spec:

```bash
$ rspec
/Users/davidpell/code/ruby/supervillains/spec/representers/supervillain_representer_spec.rb:3:in `<top (required)>': uninitialized constant SupervillainRepresenter (NameError)
```

Oops! Thanks for reminding us to create the Representer, TDD-workflow! 

```ruby
# app/representers/supervillain_representer.rb

require 'representable/json'

module SupervillainRepresenter
  include Representable::JSON
end
```

Now let's try that again...

```bash
$ rspec

  SupervillainRepresenter
maps Supervillain attributes correctly (FAILED - 1)

  Failures:

  1) SupervillainRepresenter maps Supervillain attributes correctly
  Failure/Error: expect(villain.name).to eq('Lex Luther')

  expected: "Lex Luther"
  got: nil

(compared using ==)
# ./spec/representers/supervillain_representer_spec.rb:19:in `block (2 levels) in <top (required)>'

Finished in 0.03703 seconds (files took 1.29 seconds to load)
  1 example, 1 failure
```

Our Representer definitely exists, but the `name` attribute on our supervillain isn't being mapped. To get this first test to pass, we'll need to take a look at the most basic keyword in the Representable DSL, `property`. Before doing that, however, it's worth briefly mentioning what _didn't_ go wrong here: we didn't get an `UnknownAttributeError` for `villain_id` like we did above when we tried to use `#from_json` without first mixing in the `SupervillainRepresenter`. 

One of the most basic features that Representable adds is the automatic "whitelisting" of attributes. In other words, our Representable will *only try to map the attributes that we explicitly tell it about*. Anything else that happens to be in the JSON document will be ignored. If you've worked on a Rails 4 application, this is very similar to how [Strong Parameters](http://edgeapi.rubyonrails.org/classes/ActionController/StrongParameters.html) are utilized in controllers when users create and update objects through forms.

#### Properties

We can use the `property` keyword to tell Representable to look for an attribute in the JSON doc. Let's inform our Representer about the `name` property:

```ruby
module SupervillainRepresenter
  include Representable::JSON

  property :name
end
```

Now when we run our test suite we can see that the `name` attribute is being correctly set on our supervillain object:

```bash
$ rspec

SupervillainRepresenter
  maps Supervillain attributes correctly

Finished in 0.00814 seconds (files took 2.14 seconds to load)
1 example, 0 failures
```

Great! Now let's add a test for our `first_year` property:

```ruby
# supervillain_representer_spec.rb

describe SupervillainRepresenter do 
  it 'maps Supervillain attributes correctly' do 
    # ...
    expect(villain.first_year).to eq(1940)
  end
end
```

When we run this test, we get the following failure:

```ruby
$ rspec

SupervillainRepresenter
  maps Supervillain attributes correctly (FAILED - 1)

Failures:

  1) SupervillainRepresenter maps Supervillain attributes correctly
     Failure/Error: expect(villain.first_year).to eq(1940)

       expected: 1940
            got: nil

       (compared using ==)
     # ./spec/representers/supervillain_representer_spec.rb:20:in `block (2 levels) in <top (required)>'

Finished in 0.04773 seconds (files took 3.24 seconds to load)
1 example, 1 failure
```
Here our test has failed because we need to tell `SupervillainRepresenter` about the `first_year` attribute. But just adding `property :first_year` isn't going to be enough. Why? Because unlike with `name`, the attribute that holds `1940` in the JSON document has a different name (`year_introduced`) from what we're using in our database column (`first_year`). To get around that, we need to use nicknames. 

#### As

To tell our Representer that the value we want for `first_year` is stored in the JSON document under another name, we can pass the `:as` option to `property`:

```ruby
module SupervillainRepresenter
  # ...
  property :first_year, as: :year_introduced
end
```

This says that *we* want the extracted value to be assigned to a property on our object called `first_year`, but that the value can be found in the JSON document *as* the property `year_introduced`. With that, our tests are passing again:

```bash
$ rspec

SupervillainRepresenter
  maps Supervillain attributes correctly

Finished in 0.0069 seconds (files took 1.33 seconds to load)
1 example, 0 failures
```

#### Nested Properties

How about the `publisher` attribute? This presents an interesting problem because the value we need is nested in our JSON object beneath the `publisher` attribute, and is associated with a key called `name`. 

Representable provides the `nested` keyword to help us "flatten out" a JSON document, i.e. step down into a nested structure to extract values. We can use it, along with an `as` alias, to get to our publisher name like so:

```ruby
module SupervillainRepresenter
  # ...
  nested :publisher do 
    property :publisher, as: :name 
  end
end
```

Now representable knows that it needs to 1) step down into a nested object at the JSON doc's `publisher` attribute, 2) look for an attribute called `name`, and 3) assign its value to our `publisher` attribute. And with that, our test is back to green!

#### Collections

The last of Representable's core features that I'd like to cover is the extremely powerful `collection` keyword. `collection` allows you to inform the Representer about a "one to many" relationship between the primary object you're building, and child objects that would be associated to the primary object (e.g. `has_many`, `belongs_to`). If the JSON document that you're working with contains a nested array of objects, you can build them all through the primary/parent object using the `collection` keyword. 

In our case, the `Supervillain` model `has_many :nicknames` as we saw above. The `Nickname` model would look like this:

```ruby
class Nickname < ActiveRecord::Base
  belongs_to :supervillain
end
```

Our test JSON document provides an `nicknames` attribute containing an array of nickname objects. Since there are two objects in the array, we can set up the test to count the number of nicknames associated with our supervillain after the build process:

```ruby
describe SupervillainRepresenter do 
  it 'maps Supervillain attributes correctly' do 
    # ...
    expect(villain.nicknames.size).to eq(2)
  end
end
```

This leaves us with the following failure:

```bash
$ rspec

SupervillainRepresenter
  maps Supervillain attributes correctly (FAILED - 1)

Failures:

  1) SupervillainRepresenter maps Supervillain attributes correctly
     Failure/Error: expect(villain.nicknames.size).to eq(2)

       expected: 2
            got: 0

       (compared using ==)
     # ./spec/representers/supervillain_representer_spec.rb:25:in `block (2 levels) in <top (required)>'

Finished in 0.05117 seconds (files took 1.92 seconds to load)
1 example, 1 failure
```

There are a couple small steps to get this working in `SupervillainRepresenter`, but you'll see by the end that this is absurdly simple and powerful. First we add the `collection` property:

```ruby
module SupervillainRepresenter
  # ...
  collection :nicknames, class: Nickname, extend: NicknameRepresenter
end
```

We've specified that we're going to build a collection of nicknames on our parent supervillain object (the symbol passed to `collection` must be a property in the JSON object as well as a method that the parent object responds to because of its `has_many` association, otherwise we'd need to use another `as` alias). Then we add two options, `:class` to specify that the class of object we are building in our app is `Nickname`, and that `NicknameRepresenter` should be used to parse each object in the collection. `NicknameRepresenter` doesn't exist yet, so let's make one:

```ruby
require 'representable/hash'

module NicknameRepresenter
  include Representable::Hash

  property :name
end
```

We need to use `Representable::Hash` instead of `Representable::JSON` here because each nickname object is going to be parsed into a hash by `SupervillainRepresenter` before it makes its way here. And just like that, the parent representer will leverage the child representer and build the associated objects. Our final test is green!

```ruby
$ rspec

SupervillainRepresenter
  maps Supervillain attributes correctly

Finished in 0.02632 seconds (files took 1.54 seconds to load)
1 example, 0 failures
```

And here's the final state of our `SupervillainRepresenter`:

```ruby
require 'representable/json'

module SupervillainRepresenter
  include Representable::JSON

  property :name

  property :first_year, as: :year_introduced

  nested :publisher do 
    property :publisher, as: :name 
  end

  collection :nicknames, class: Nickname, extend: NicknameRepresenter
end

```

This has been a whirlwind tour of Representable's main keywords. As you can see, Representable is a really powerful tool to simplify the process of mapping data from an external source to your application's ActiveRecord models. There is much more that could be said, such as the fact that Representable works just as well in the other direction (rendering your AR objects as custom JSON or XML objects). There are also powerful customization options for processing the data you get before it is saved to your database, setting up build strategies for collection objects (e.g. initialize and overwrite existing objects instead of create new ones), etc. I'll save those for another time!
