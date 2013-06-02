# Darwinjs

Darwin is a javascript framework for people that take error
handling seriously and want to achieve it through progressive
enhancement and graceful degradation.

Darwin will also let developer write clean and encapsulated
code that encourages self documentation.

## Installation

Add this line to your application's Gemfile:

    gem 'darwinjs-rails'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install darwinjs-rails

## Getting Started

First, as one time configuration, add autoloader in your
application.coffee file :

```coffee
#= require 'darwin'
#= require_tree ./views
#= require_tree ./controllers

$(->
  Darwin.Loader.run()
)
```

You can now generate a javascript module using the provided
generator :

```
$ rails generate darwin:assets users/index

      create  app/assets/javascripts/controllers/users.coffee
      create  app/assets/javascripts/views/users.coffee
      create  app/assets/javascripts/controllers/users/index.coffee
      create  app/assets/javascripts/views/users/index.coffee
```

This will create your controller and your view in the `users` namespace.

Now add a `data-module` attribute in your users index view to
autoload your module :

```erb
<div id="users" data-module="User.Index">
  <ul>
  <% @users.each do |user| %>
    <%= render 'user', user: user %>
  <% end %>
  </ul>
</div>
```

This will automatically initialize your module.

A module is composed of two files :

* a controller that handles events
* a view that handles DOM manipulation

Here is a typical view :

```coffee
class App.Views.Users.Index extends Darwin.Controller
  @options {
    selectors:
      show_users: 'a#show_users'
      user_block: '#users'
      user:
        'sel': '.user'
        more: '.more a'
        delete: 'a[data-method="delete"]'
  }

  show_info_for( $link ) ->
    $link.next( '.info' ).show()

  remove_user_for( $link ) ->
    $link.parent().remove()
```

And the corresponding controller :

```coffee
class App.Controllers.Users.Index extends Darwin.Controller
  @options {
    View: App.Views.Users.Index

    events:
      'Toggle user block':      { el: 'show_users', type: 'click' }
      'Show user info':         { el: 'user_more', type: 'click' }
      'Delete user on server':  { el: 'user_delete', type: 'click' }
  }


  show_users_clicked: ->
    @view.get( 'user_block' ).fadeIn()


  user_more_clicked: ( $link ) ->
    @view.show_info_for( $link )

  
  user_delete_clicked: ( $link ) ->
    if confirm( 'Really delete user ?' )
      $.get( $link.attr( 'href' ), =>
        @view.remove_user_for( $link )
      )
```

As you see, a view acts as single point of configuration for selectors.
Any change made then reflect to the whole javascript codebase.

In the same way, controller acts as a single point of configuration
for events. You can tell what a module does looking at the first
lines of the controller file.

But there is more happening under the hood, here. First, all you DOM
elements retrieved by view selectors are cached. Upon further call
they are retrieved without hitting the DOM again, which is very
costly in term of performances.

Furthermore, all event callbacks are wrapped so that they do not
execute if an error occured. In case of error, events are simply
deactivated and any link is followed, reloading the page and letting
server side handle what has to be done, so your user doesn't even
notice something got wrong.

Ready for more ? See [introduction](doc/introduction.md).


## Philosophy

If I had to choose a catchphrase for darwin, it probably would be :

    Darwin knows client side isn't server side.


### You don't execute any code

There is something very cool with server generated pages : if it works for
you, it works for everyone. If something goes wrong, you probably receive
mail about the exception (you do, right ?). We have all of this for free,
because there is a single or a small set of machines doing the work.

Now, consider who is doing the work with a javascript codebase. It's your
visitors. That's really cool : charge on server is lowered and you don't
need to wait for the network to do stuff.

But you have as much execution environments as you have visitors. We could
even say you have as much execution environments as you have visits, because
you can't expect your visitor to have a stable and coherent environment.
What's the state of memory usage of your visitor ? Is his bandwith saturated
by multiple parallel downloads ? What extensions his browser is running ?

And now, the true question : what happens to your page when an error occurs ?

Links having "#" as href are a plea : if your javascript crashed, they will
simply do nothing. Think of what people do when they click a link that does
nothing. If you're a developer, you probably will instinctively reload the
page. If you're a lambda user, you probably get frustrated, yell "it doesn't
work !" and go somewhere else. How do we solve that ?

Darwin suggest that we do so through progessive enhancement and graceful
degradation. Your application should work perfectly without javascript. When
javascript runs, it enhances your interface to add all its cool features.
If something wrong occurs, interface is reverted back to its initial state,
and event callbacks are deactivated, so that feature works again and
promises made to user are fullfilled. I call that server side fallback.


### Your js application is an interactive presenter

On server side, we like to think about our application as a collection of
resources. Resources on the lower level are mapped to database tables.
Then we have model that abstracts operations on table data. Then we have
controllers that map requests on resources. And finally, we have views
that handles presenting our UI.

Should we use this same approach for client side ? In a previous version,
darwin did have a Model class. This turned out to be unintuitive and globally
useless : most operations on data (computation, translations, etc) are
already done by the server side. The model class was only called by
controllers to fire requests and then immediately pass data as callback.
Well, `$.getJSON` and `$.post` handle that pretty well. And since you conceived
your application through progressive enhancement and graceful degration,
chances are there's a form somewhere containing all the data and ready to
be serialized when lot of "model" data have to be saved.

But anyway, this does even not makes sense. Your server side app is built
around a database. Your client side app is built around html data. It's
a presenter that handles user feedback.

What we really need is a layer that handles DOM manipulation, and a layer
that handles interruptions (events and requests). They are the view layer
and the controller layer.

A view handles DOM manipulation. It abstracts selectors so that if anything
change in your HTML, you won't have to change it all over your codebase,
but just in one place. This acts as a single point of configuration for
retrieving DOM elements. In a view, you add all methods that do
special DOM manipulation, and especialy the setup (progressive enhancement)
and tear down (graceful degradation) code.

A controller handles events and requests. A configuration method is
used at the top of controller with descriptive comment keys, so that
you can see at a glance what controller does. Callback names are infered
from event descriptions, and they are all deactivated if a crash occurs.

Thus, view handles the presenter part, and controller handles interactivity.


### You may have several features on a page

Again, on server side, we love to think of application in terms of resources.
But on client side, it's difficult to stick to that pattern : we often
use a composition of resources to provide a feature. You may have a list
of users who you want to be able to create / edit / update / delete and
you can stick to single resource feature. But you may also want to display
a list of customer - a special kind of user - and to display for each their
last purchases. Now, you also want to do nifty javascript stuff on those
purchases, which are an other resource.

Instead of having a big javascript controller that handles things pretty
unrelated, darwin let you decompose a page in features with modules.

A module is a controller and a view bound to a subset of your html page.
The root element acts as a sandbox for your module : controller and view
can't access anything outside it. So you can have a module on the whole
page if it's a single feature page, or you can sandbox your module to a
given part of the DOM and have multiple modules, you can even have a
module sandboxed to a part of the DOM which is contained in the root element
of an other module. This ensures your modules respect single responsibility
principle.

And of course, modules are able to communicate with each other if needed,
through class events : any controller or view can fire and bind events,
just like you would do with a DOM element.


## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
