# Consume JSON API on Rails

There are several guides on how to serve JSON API on Rails, but none is really on consuming. So, this guide will fill up the gap for a bit.

```json
{
  "data": [
    {
      "id": "1",
      "type": "posts",
      "attributes": {
        "title": "2016-06-13",
        "body": "Today is Monday"
      }
    },
    {
      "id": "2",
      "type": "posts",
      "attributes": {
        "title": "2016-06-14",
        "body": "Today is Tuesday"
      }
    }
  ]
}
```

Let's setup a Rails application to consume the JSON API.

```bash
$ rails new json-api-consumer --skip-test --skip-spring --skip-active-record
```

We skip `spring` to get rid of some unnecessary known troubles dealing with it.
We skip `ActiveRecord` since our resources are from the JSON API server.

We'll be using `gem 'json_api_client'` instead of `gem 'activeresource'`, `gem 'jsonapi_resource'`, etc.

```ruby
# Gemfile

gem 'json_api_client'
```

To get a list of posts, we create a `Post` resource as well as a `BaseResource` that will inherited by all resources.

```ruby
# app/model/base_resource.rb

class BaseResource < JsonApiClient::Resource

  # root url to your API server, you may set it in development.rb or environment variable
  self.site = 'http://localhost:3000'

end
```

```ruby
# app/model/post.rb

class Post < BaseResource; end
```

Then create a `PostsController` and its associated views.

```bash
$ rails g controller posts
```

```ruby
# app/controllers/posts_controller.rb

class PostsController < ApplicationController

  def index
    @posts = Posts.all
  end

end
```

```slim
// app/views/posts/index.html.slim

= render @posts
```

```slim
// app/views/posts/_post.html.slim

h1 = post.title
p = post.body
```

Now, start your JSON API server & client.

```bash
server$ rails s webrick -p 3000
client$ rails s webrick -p 3001
```

Browse to `http://localhost:3001` and you shall see a list of posts.

## Authentication

Let's just allow authenticated user to see the list of posts.

On the JSON API server, we use token authentication. It expects an `auth_token` and its corresponding `email` inside the request header.

```ruby
# app/controllers/api/v1/base_controller.rb

class Api::V1::BaseController < ApplicationController

  before_action :authenticate_user!

  private

  def current_user
    @current_user
  end

  def authenticate_user!
    auth_token, options = ActionController::HttpAuthentication::Token.token_and_options(request)

    user = User.find_by(email: options&.dig(:email))

    if user && ActiveSupport::SecurityUtils.secure_compare(user.auth_token, auth_token)
      @current_user = user
    else
      render nothing: true, status: 401
    end
  end

end
```

then we change the `PostsController` to inherit from `Api::V1::BaseController` instead of `ApplicationController`.

```ruby
# app/controllers/api/v1/posts_controller.rb

class Api::V1::PostsController < Api::V1::BaseController

  ...

end
```

Now, if you try to request without the `Authorization` inside the header, it will fail with status `401 Unauthorized`.

So, how do we add `Authorization` inside the request header while using `JsonApiClient::Resource`? In this case, we'll need a faraday middleware to inject the user's `email` and `auth_token` into request that requires authentication. The author of `JsonApiClient` has an example for this. Refer [this issue](https://github.com/chingor13/json_api_client/issues/64#issuecomment-116761917) for more details. The implementation of the faraday middleware basically looks like this:

```ruby
# app/middleware/token_auth_middleware.rb

class TokenAuthMiddleware < Faraday::Middleware

  attr_reader :klass, :app

  def initialize(app, klass)
    super(app)
    @klass = klass
  end

  def call(environment)
    environment[:request_headers]['Authorization'] = "Token token=#{@klass.auth_token}, email=#{@klass.email}"
    @app.call(environment)
  end

end
```

Then inside the `BaseResource`, we need to pass the `auth_token` and `email` to the middleware.

```ruby
# app/model/base_resource.rb

class BaseResource < JsonApiClient::Resource

  ...

  class << self

    attr_writer :email, :auth_token

    def email
      @email ||= begin
        superclass.email if superclass.respond_to?(:email)
      end
    end

    def auth_token
      @auth_token ||= begin
        superclass.auth_token if superclass.respond_to?(:auth_token)
      end
    end

    def connection
      super.tap do |conn|
        conn.use TokenAuthMiddleware, self
      end
    end

    def authenticate_with(email, auth_token)
      self.email = email
      self.auth_token = auth_token
      yield
    ensure
      self.email = nil
      self.auth_token = nil
    end

  end

end
```

Now we can use the `BaseResource#authenticate` inside `ApplicationController` to perform any resource request that requires authentication.

## [Extra] Sessions Controller & View

```ruby
# app/controller/sessions_controller.rb

class SessionsController < ApplicationController

  skip_around_action :authenticate_user!, only: [:new, :create]

  def new
    @session = Session.new(default_params)
  end

  def create
    @session = Session.new(session_params)

    if @session.save
      store_credentials
      redirect_to my_dashboard_path
    else
      flash.now[:error] = 'Login failed!'
      render :new
    end
  end

  private

  def session_params
    params.require(:session).permit(:email, :password)
  end

  def default_params
    { email: '', password: '' }
  end

  def store_credentials
    session[:auth_token] = @session['auth-token']
    session[:email] = @session['email']
  end

end
```

```slim
// app/views/sessions/new.html.slim

.row
  .col-sm-12.col-md-4.col-md-offset-4

    h3 Sign in

    = simple_form_for @session, url: session_path do |f|

      = f.input :email, class: 'form-control'

      = f.input :password, class: 'form-control'

      = f.submit 'Sign in', class: 'btn btn-primary'

```

Assuming that we already have a `SessionsController` that will initiate a session with the JSON API server, and we store the returned `email` and `auth_token` into a `session` on Rails.

```ruby
# app/controllers/application_controller.rb

class ApplicationController < ActionController::Base

  around_action :authenticate_user!

  rescue_from JsonApiClient::Errors::NotAuthorized, with: :not_authorized!

  private

  def authenticate_user!
    BaseResource.authenticate_with(session[:email], session[:auth_token]) do
      yield
    end
  end

  def not_authorized!
    redirect_to new_session_path, flash: { error: 'Access denied!' }
  end

end
```

You can try to perform the authenticated & unauthentiated request using [Postman](https://www.getpostman.com/). For authenticated request, you need to set `Content Type` to `application/json` & `Authorization` to `Token token=<auth-token>, email=<email>` at the request headers.

Now if you try to do an unauthenticated request to `GET api/v1/posts`, you should get a `401 Unauthorized` response, and you'll get a list of post in return if you are authenticated.

## Pagination

Pagination is very important for a listing view. Let's implement one. It's very simple on the server side. With `ActiveModelSerializers`, we can use `kaminari` or `will_paginate` like how we use them normally. The JSON API response for paginated resources will have extra information about the pagination which looks like this:

```json
{
  "data": [
    {
      "id": "1",
      "type": "posts",
      "attributes": {
        "title": "2016-06-13",
        "body": "Today is Monday"
      }
    },
    {
      "id": "2",
      "type": "posts",
      "attributes": {
        "title": "2016-06-14",
        "body": "Today is Tuesday"
      }
    }
  ],
  "links": {
    "self": "http://localhost:3000/api/v1/posts?page%5Bnumber%5D=1&page%5Bsize%5D=25",
    "next": "http://localhost:3000/api/v1/posts?page%5Bnumber%5D=2&page%5Bsize%5D=25",
    "last": "http://localhost:3000/api/v1/posts?page%5Bnumber%5D=15&page%5Bsize%5D=25"
  }
}
```

We can't use the provided pagination links if the request requires authentication. So, we need to extract the pagination information like, current page number, first page number, previous page number, next page number and last page number if there is any. Then we build paths back to our `index` with query params `page` & `per_page` and perform the authenticated request from our controller to the server.

```ruby
# app/presenters/paginator.rb

class Paginator

  PAGE_NUM_KEY = 'page[number]'
  PER_PAGE_KEY = 'page[size]'
  PAGER_SEQUENCES = [:first, :prev, :next, :last].freeze

  attr_reader :collection

  def initialize(collection, controller)
    @collection = collection
    @controller = controller
  end

  def pager_urls
    (PAGER_SEQUENCES & page_links.keys).each_with_object({}) do |seq, urls|
      urls[seq] = @controller.url_for(page: query_params(page_links[seq])[PAGE_NUM_KEY].first.to_i)
    end
  end

  private

  def query_params(url)
    CGI.parse(URI(url).query)
  end

  def page_links
    @collection.pages.links.symbolize_keys
  end

  def per_page
    @collection.pages.per_page
  end

end
```

```ruby
# app/controller/posts_controller.rb

class PostsController < ApplicationController

  def index
    posts = Post.paginate(page: params[:page], per_page: params[:per_page]).all
    @paginator = Paginator.new(posts, self)
  end

end
```

Then, we create a helper to generate a bootstrap pager from `@paginator`.

```ruby
# app/helpers/application_helper.rb

module ApplicationHelper

  def pager(paginated)
    content_tag(:nav) do
      concat(content_tag(:ul, class: 'pagination') do
        paginated.pager_urls.map do |seq, url|
          concat(content_tag(:li) do
            content_tag(:a, seq, href: url)
          end)
        end
      end)
    end
  end

end
```

Next, inside the views,

```slim
// app/views/posts/index.html.slim

= render @paginator.collection
= pager(@paginator)
```

Make sure you have lots of data to check if the pagination is working. By default, it's 25 records per page.
