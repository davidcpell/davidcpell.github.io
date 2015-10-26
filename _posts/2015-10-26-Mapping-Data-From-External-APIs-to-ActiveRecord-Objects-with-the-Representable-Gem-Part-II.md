---
layout: post
title: Mapping Data from External APIs to ActiveRecord Objects with the Representable Gem - Part II
---

### Requests and Responses

In [Part I][1] of this series I covered some basic concepts related to the question, "What is an API?" As a brief refresher: the jargon-y term "API" (Application Programming Interface) refers to an object's vocabulary, by which I mean the collection of all messages (methods) to which the object is able to respond. When we send messages to objects that they don't understand, we get a `NoMethodError`. Remember also that "API" can be used more broadly to refer to a group of related objects and all of their vocabularies, and that network applications often "expose" some of these objects so that we can interact with them from the outside (see below).

If you have a little bit of Rails experience under your belt you're already familiar with objects in your applications sending messages to each other, even if you've never thought about it in those terms:

```ruby
class SupervillainController < ApplicationController
  def create
    @villian = Supervillain.new(villain_params)

    if @villain.save
      # yada yada
    else
      # yada yada
    end
  end
end
```

In the above example there are two objects in play: an instance of `SupervillainController` and a new instance of the `Supervillain` class. They're having a conversation by sending messages back and forth. The controller sends the `.new` message to the `Supervllain` class, and `Supervillain` responds with (i.e. returns) a new instance of itself, which is saved to the variable `@villain`. The controller then sends the `#save` message to that instance. Assuming there are no validation errors, the `@villain` instance responds with `true`, which the controller uses to decide what it will do next (`if...else...end`).

#### Showing Data

In a standard Rails application the kind of conversation outlined above is kicked off "from the outside," most likely via a user's form submission. Similarly, if we wanted to _view_ a resource (rather than create one), we might click on a link that would be routed to the `#show` controller action:

```ruby
class SupervillainController < ApplicationController
  def show
    @villain = Supervillain.find(params[:id])
  end
end
```

Here the controller sends the `.find` message to our `Supervillain` class, and an instance of `Supervillain` is returned before the controller renders the views (the class might also respond with `nil` if there was no Supervillain in the database with the given id).

Ok, but what if we need to "show" and make use of the data from a larger number of records than would be practical for a person clicking on links? In [Part I][1] we already discussed how visiting a webpage and then manually creating our own database records one by one would be impractical. If only there were an easy way to request many records at once, in a format that would allow us to automate the process of receiving it, processing it, and saving it to our own database.

#### The Web is Text

As you may have guessed, what I outlined in the last paragraph is the core topic of this series. Creating, reading, updating, and deleting data on the internet is all about sending requests and receiving responses (similar to the way in which objects within an application send messages and receive return values). Your web browser is one tool for sending requests for data over the internet. When you type `davidpell.net` in your address bar and press enter, a request goes out to the server where my blog is hosted, a response is returned in the form of a *text* file. The text is formatted in a way that you know as HTML, which your browser knows how to read and transform ("render") into what you see on the page. Your browser can also receive text files formatted as CSS and Javascript and deal with them accordingly.

There are other tools that allow us to send requests and receive text responses. If you're on a Mac, open your terminal and type `curl davidpell.net`. The big block of text that will appear is the same HTML that your browser gets - your terminal just doesn't know what to do with it besides print it out as text. `curl` (technically `cURL` if you were to look it up) is doing something very similar to what happens behind the scenes when you type a URL into your browser.

#### JSON Endpoints

We can think of the URLs that we type into our web browsers or pass to `curl` as "endpoints." The URL itself, with its domain (everything up to .com), path (stuff between forward slashes), and parameters (the hard-to-read stuff you sometimes see at the end of a URL following a question park and separated by &), are the request's map to the destination (endpoint). Some applications set up endpoints that are meant to serve the kind of use case we've been discussing. Rather than HTML, these endpoints usually return data in a format such as XML or JSON, with the latter being the one we care about right now.

JSON means "Javascript Object Notation." Don't worry too much about why it's called that for now. As a Rails developer, you could think of it as a hash where all the keys are strings and the values are formatted in such a way that they can easily be parsed by pretty much any client language. Since it comes to us as text, it's really one big string that we can use Ruby's `JSON` library to parse into a hash.

Let's take a look at a response from The Internet Chuck Norris Database as an example of a simple JSON object:

```
david $ curl http://api.icndb.com/jokes/random
{ "type": "success", "value": { "id": 129, "joke": "Chuck Norris keeps his friends close and his enemies closer. Close enough to drop them with one round house kick to the face.", "categories": [] } }
```
Here the object in question is a "joke." If we look past the "type" key, which just tells us that the request was successful, we can see the joke object inside of the "value" key. In the host database, a joke is represented as an object with three attributes: an id, the actual joke text (labeled "joke"), and a set of categories that it belongs to.

If the joke object were a hash, we could easily pass it to the `#new`, `#create`, or `#update` methods provided by the ActiveRecord API. Creating a Ruby hash from a JSON object is simple enough: the `JSON` library has a `.parse` method that does it very simply. However, there are some problems with the JSON object as it's returned that make it less than ideal for mapping directly to an ActiveRecord object. How do we deal with the fact that the real joke object is actually nested within the "value" key? What if our `jokes` table has a column named `body` or `text` for storing the text of the joke, whereas the JSON has a key called "joke"? In the last part of this series I'll finally get around to introducing the Representable gem and covering some of the basic tools that it provides for making it very easy to solve those problems.

[1]: http://bit.ly/1MPslCO
