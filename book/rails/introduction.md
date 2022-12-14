# Introduction to our example application and setup

## Example application

We will use a fake example app, Humon, to explain and show the concepts
throughout this book. Humon is an app that lets you find nearby events.

In the Humon application, a user can have many events as an event owner. An
event has geolocation information (latitude and longitude), which allows us to
plot it on a map. A user has and belongs to many events through attendances. A
user can have only one attendance per event.

![Humon database representation](../images/humon-database-representation.png)

The Humon application never asks for a username or password. Instead, we will
assign an auth token to all new devices using our API. The mobile device handles
storing this token and signing all requests. This approach only allows one user
on a single phone. They can start using the application immediately, though.

We wanted to create the most straightforward application possible. This led us
to choose immediate usability over a more complex authentication system.

We will provide code snippets in context. You can also view the [entire example
application in GitHub][] to see how we have structured it.

## Setting up our project

We started our project using [Suspenders][]. Suspenders is a Rails 6 template
with thoughtbot's standard defaults. Creating a Rails app with Suspenders is
simple. All you have to do is follow the instructions in the `README`.

Using Suspenders is optional when following this book. It includes all the gems
we will use to test-drive our API. It has [factory_bot][], [RSpec][], [Shoulda
Matchers][], and so much more. If you start your project without Suspenders, add
those gems to your `Gemfile`.

## Parsing incoming JSON requests

[MultiJson][] is an adapter for JSON parsers. Another adapter familiar to Rails
developers is ActiveRecord. ActiveRecord provides a standard interface to
database drivers like Postgres and MySQL. MultiJson offers a friendly
interface for JSON parsers like Oj and Yajl. We get MultiJson for free with
Rails. This is because MultiJson is a dependency of ActiveSupport.

For parsing JSON, we chose the Oj gem. To use the Oj gem in your Rails
application, add it to your `Gemfile` and install it with the `bundle install`
command. We chose Oj because it is a fast JSON parser. From the MultiJson
README:

> When loading, libraries are ordered by speed. First Oj, then Yajl, then the
> JSON gem, then JSON pure. If no other JSON library is available, MultiJSON
> falls back to OkJson, a simple, vendorable JSON parser.

## Generating outgoing JSON responses

There is no shortage of methods to render a JSON response. We looked into
[Active Model Serializers][], [RABL][], and using the Rails `.as_json` method.
We chose [Jbuilder][] for rendering JSON because of its excellent community
support. The consistency of its view logic to other kinds of Rails views, such
as Haml, ERB, and Builder, was also a factor.

With Jbuilder, we render the JSON from Rails controllers like any other view. We
can use partials as with any other Rails view template to compose JSON. Also,
`cache!` has the same method signature as
[`ActionView::Helpers::CacheHelper`][]. (It uses `Rails.cache` under the hood.)
We will delve into the details of views and caching in later chapters.

## Versioning

Before building our API, we must consider how we will handle versioning. Web
developers can deploy as often as they want. Users see the deployed code with
every browser page refresh.

Mobile developers have a lag time before their app store approves an app's new
version. There is also a lag time before users update to the latest available
application version.

Mobile apps use the same API endpoints until the user downloads the latest
release. You want to ensure you support users with older versions of your mobile
application. To do so, you must maintain the same general JSON data structures
on the backend for those users.

You will discover new and better ways of structuring your JSON responses. This
will happen as time progresses and your application grows. Once that happens,
the easiest way to support old app versions is to release a new API version.
This will also allow you to add new versions with different JSON structures.

Releasing many versions of an API is outside this book's scope. Ryan Bates has
an excellent [RailsCast][] on this topic. We will future-proof our API by
including an `api/v1` subdirectory. Our routes file looks like this:

```ruby
# config/routes.rb
Humon::Application.routes.draw do
  scope module: :api, defaults: { format: 'json' } do
    namespace :v1 do
      # resources will be here
    end
  end
end
```

The API is now scoped via the URL. For example, with our setup above, we will
end up with the following endpoint. This is the path we will use to request a
single event at version 1 of the API.

```ruby
"#{Rails.root}/v1/event/:event_id"
```

## API Documentation

In the early days of your JSON API, you will likely change the data returned and
the data structure almost daily. Communication is crucial and challenging for
all software development teams. It can be challenging when working across groups
that speak different programming languages. Both Rails developers and mobile
developers talk "JSON". We found it challenging to ask mobile developers to stay
updated with API changes via GitHub.

A solution we found for keeping all developers in sync was using GitHub's wiki
feature. We use this as a source of API documentation. Updating the wiki after
each API change required a small amount of work for our Rails developers. Having
somewhere mobile developers could find up-to-date API documentation was an
invaluable resource. You can see how we structure it in our [wiki][].

If exploring other documentation options interests you, here are some
suggestions:

- [fdoc](https://github.com/square/fdoc)
- [apipie-rails](https://github.com/Pajk/apipie-rails)
- [YARD](http://yardoc.org/)

## API Security

APIs built for commercial use usually have some concept of a client id and/or
client secret. These strings are unguessable. They act as a username and
password combination required for all API requests. Requiring a client id and
secret ensures that only known users can access the API. This also allows the
API to turn off access to a particular person or application. If their usage
violates the API terms of service, we can disable their key. [This blog post on
API security][] explains the benefits of API keys.

Humon is an API built for non-commercial purposes. We can create a simple
permission scheme without many API tokens. But, we want to ensure that only some
people can query our API. Having zero security would mean anyone could create a
`curl` request to an endpoint. They could hit our database and cause unknown
damage.

As a security measure, we need a header of `tb-app-secret` for the `POST users`
request. We ensure that only those with a value matching the Rails app's can
create new users via the Humon API. We store the app secret in an environment
variable to exclude it from version control.

We use [dotenv][] so that our API can read environment variables from a `.env`
file while in development mode. We also add `.env` to `.gitignore` to exclude it
from our git repository. If the app secret sent by the client on `POST users`
differs from the API secret, it returns a "404 Not Found" error.

For all requests (except for a `POST users` request), we need the header to
contain a `tb-auth-token`. During a `POST users` request, we create an auth
token for a user and return it in the response JSON. The mobile app stores that
token and sets it in the header of every following request.

You can see how we implemented an app secret and auth tokens for POST users.
Look at the following.

- [`before_filter` in our `UsersController`][]
- [`before_validation` in our `User` model][]

[`actionview::helpers::cachehelper`]: http://api.rubyonrails.org/classes/ActionView/Helpers/CacheHelper.html
[`before_filter` in our `userscontroller`]: https://github.com/thoughtbot/react-native-on-rails/blob/main/example_apps/rails/app/controllers/api/v1/users_controller.rb
[`before_validation` in our `user` model]: https://github.com/thoughtbot/react-native-on-rails/blob/main/example_apps/rails/app/models/user.rb
[`dotenv`]: https://github.com/bkeepers/dotenv
[`rails.cache`]: http://guides.rubyonrails.org/caching_with_rails.html#cache-stores
[active model serializers]: https://github.com/rails-api/active_model_serializers
[example application in github]: https://github.com/thoughtbot/react-native-on-rails/tree/main/example_apps/rails
[factory_bot]: https://github.com/thoughtbot/factory_girl_rails
[jbuilder]: https://github.com/rails/jbuilder
[multijson]: https://github.com/intridea/multi_json
[rabl]: https://github.com/nesquena/rabl
[railscast]: http://railscasts.com/episodes/350-rest-api-versioning
[rspec]: https://github.com/rspec/rspec-rails
[shoulda matchers]: https://github.com/thoughtbot/shoulda-matchers
[suspenders]: https://github.com/thoughtbot/suspenders
[this blog post on api security]: https://stormpath.com/blog/top-six-reasons-use-api-keys-and-how/
[wiki]: https://github.com/thoughtbot/react-native-on-rails/wiki
