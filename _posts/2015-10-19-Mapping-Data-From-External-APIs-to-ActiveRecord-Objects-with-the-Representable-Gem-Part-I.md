---
layout: post
title: Mapping Data from External APIs to ActiveRecord Objects with the Representable Gem - Part I
---

## What is an API?
At my day job I spend a lot of time writing code to communicate with APIs and retrieve data from them. What exactly does that mean? First we should take a step back and look at how we get data into our databases.

Let's say we're working on a Rails app related to comic book characters and have a model like this:

```ruby
# app/models/supervillain.rb

class Supervillain < ActiveRecord::Base
  has_many :plots
end
```

...and the `supervillains` table in our database has columns for `name`, `alias`, and `publisher`.

#### Creating data in the Rails console
One way for us to populate the database with supervillain entries would be to create entries by hand in a Rails console:

```ruby
Supervillain.create(name: "Lex Luther", alias: "Mockingbird", publisher: "DC Comics")
Supervillain.create(name: "Magneto", alias: "Magnus", publisher: "Marvel Comics")
... # this could take a while...
```
That's obviously not a good idea if we have a lot of data to import.

#### User-submitted data (forms)

Another way we could populate our db is by having forms on a web page that can be filled out by users. When a user submits our imaginary Supervillain form, the data is posted to the `create` action of `SupervillainsController`, which does the work of saving the new Supervillain object to the database.

#### A better option

Databases can also be programmatically "seeded" with large amounts of data from various third-party sources. In other words, we let someone else collate all the data and then we go and plunder it! Or sometimes they send it to us. This could happen in a number of ways, such as parsing and importing the contents of CSV files (comma-separated values, a text file that can be read as a spreadsheet with commas delimiting the columns) and, most importantly for this post, retrieving the data in JSON format from an external API.

#### What is an API?

"API" is one of those terms that you've probably heard thrown around a lot if you're just getting started with software development. Maybe you've done some Googling or even visited [Wikipedia][2] to try to get a better understanding of what people are talking about. If you _haven't_ heard it yet, I assure you that you _will_ hear people talking about APIs sooner or later.

Before making my career change to software development, I was a foreign language teacher. Because of my background with learning and teaching languages I like to think of APIs as the vocabularies "spoken" by the objects in an application. For example, all of the methods that a string can respond to, taken collectively, could be referred to as the String's vocabulary - its API. This understanding of API as vocabulary fits well with the convention of referring to Ruby methods as "messages" that objects send to and receive from each other popularized by [Sandi Metz][3] and others. When you build your own classes, you define their APIs as you specify which messages they are able to respond to. So let's open our class back up:

```ruby
# app/models/supervillain.rb

class Supervillain < ActiveRecord::Base
  has_many :plots

  def maniacal_laughter
    "MUAHAHAHA"
  end
end
```

Now we can open up our console again and play around like this, with a method we've defined and one we haven't:

```ruby
[1] pry(main)> villain = Supervillain.new
=> #<Supervillain:0x007fb29ba24ab8>
[2] pry(main)> villain.maniacal_laughter
=> "MUAHAHAHA"
[3] pry(main)> villain.feel_pity
NoMethodError: undefined method `feel_pity' for #<Supervillain:0x007fb29ba24ab8>

# A NoMethodError means, "I don't know what you're talking about! That message isn't in my vocabulary!"
```

When we go back to look at the technical definitions of API, it becomes more clear what the "I" ("interface") means. Interfaces act as the meeting point between two parties and facilitate their communication or interaction. Your keyboard and mouse are the interface that allow you to provide instructions to your computer. Language and the alphabet are interfaces that are allowing me to communicate all of this information to you. A Ruby application's interface is the set of objects of which it is comprised and the messages to which they respond.

#### Network APIs

For applications configured to operate over network connections (like your web apps!), API can refer specifically to the objects/resources that its developers have "exposed" to users to be created, read, updated, and destroyed. In part 2 of this series I'll look at what it means to make network requests to an external API and receive a response, which can then be processed and used in some way by our own application.




[1]: https://github.com/apotonick/representable
[2]: https://en.wikipedia.org/wiki/Application_programming_interface
[3]: https://twitter.com/sandimetz?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor
