# Authentication with a Rails API
_Session management with a familiar tool -- Rails's session!_

[Rails JSON API] backend?
JS frontend?
Want users to be able to sign up, log in, and log out?  Maintain their session even after closing browser tab?

Criteria for this blog to be applicable:

- Rails JSON API backend built with `--api` flag
- Some kind of JS frontend that sends AJAX requests (could be Vanilla JS, React, or some other JS framework)
- JS frontend must have an identifiable domain that can be whitelisted

No time?  Just want the quick setup?  OK. Here's the short version:
## QUICK CONFIG GUIDE

In your `Gemfile`:

Add or uncomment `rack-cors`.

Add or uncomment `bcrypt` if you want secure passwords (and you do).

Run `bundle install`.

Head to `config/initializers/cors.rb` and uncomment the commented-out code.

Change `origins 'example.com'` to `origins 'www.yourdomain.com'`.  Obviously, put in your domain.

Add `credentials: true` to the `resource` call, so:
```ruby
resource '*',
  headers: :any,
  methods: [:get, :post, :put, :patch, :delete, :options, :head],
  credentials: true
```

Add these two lines of code to `config/application.rb` inside the `Application` class:
```ruby
config.middleware.use ActionDispatch::Cookies
config.middleware.use ActionDispatch::Session::CookieStore, key: '_cookie_name', expire_after: 14.days, httponly: true
```

Add to `ApplicationController`:

`include ActionController::Cookies`

Now, with every AJAX request, add `credentials: "include"` in the options, such as:

```js
fetch("http://localhost:8000/users/12/secrets", {
  method: "GET",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    "Accept": "application/json"
  }
})
  .then(response => response.json())
  .then(doAwesomeThingsWithYourSecrets)
```
And that's it!  Now your Rails responses will include an HTTP-only cookie with session info.  And each AJAX request with `credentials: "include"` will send that cookie back to be authenticated.  To log in, add a unique, identifying key/value pair to `session` such as `session[:user_id] = @user.id` in your login controller action after confirming correct login credentials.  Add session helper methods such as `#current_user` and `#logged_in?` to your `ApplicationController` if you like.

To log out, `session.clear`.

For authorization, protect your controller actions.  For example, in `SecretsController`:
```ruby
  def show
    @secret = Secret.find(params[:id])
    if @secret.user == current_user
      render json: @secret.to_json(only: [:content, :id]), status: :ok
    else
      render json: {
        error: "Those aren't your secrets!"
      }, status: :unauthorized
    end
  end
```
Easy peasy!

## SLIGHTLY MORE DETAILED GUIDE
_Follow along to build a Rails backend that can handle auth.  Up to you to provide the frontend of your choice._

Run `rails new secrets-backend --api`
Of course, "secrets-backend" is the name of your app.

The `--api` flag removes a bunch of middleware from the generation of the Rails application.  It also changes the configuration of some of the generators.  We are saying, "Hey Rails, how's it going?  For this app you can take it easy -- no need to include middleware for views since we're just going to be returning JSON."

If we want auth, though, it removes a little _too much_ middleware... more on that later.. Let's continue:

We know we're going to want our Rails server to process AJAX requests from external domains, which means we need to enable [Cross Origin Resource Sharing], or CORS.  In the `Gemfile`, comment in the line that says
`gem 'rack-cors'`
Then run `bundle install` again.

Enabling the `rack-cors` gem is not enough -- we also need to adjust the CORS configuration.  So let's open `config/initializers/cors.rb`.  First, uncomment this code:
```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'example.com'

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

On the line that starts with `origins`, we're telling Rails what domains to accept AJAX requests from.  Let's start by replacing `'example.com'` with the the catch-all character `*`.  This means, "Hey Rails, please accept AJAX requests from all domains."  So now it should look like this:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

Keep this file open.  We're going to make more CORS configuration changes soon.  We should also pause for a moment to ensure understanding around the term AJAX.  The way it's being used in this blog, AJAX refers to requests being sent to our backend from our JS code.  AJAX stands for [Aysychronous JavaScript and XML], but at this point should really be AJAJ, since we're getting JSON responses instead of XML responses from our Rails server.  Some might distinguish AJAX and the [Fetch API], but we're clomping them together here.  Bottom line, when we say "make a fetch request" and "make an AJAX request" we mean the same thing in this blog.  Please do not confuse with [jQuery's `.ajax()`] method, although that would fall into the same category.  Sheesh.

With our basic CORS config setup, let's pause and scaffold out a user model we can test with.  Since this is a blog on auth, let's give users email and password attributes.  Let's use the `bcrypt` gem to hash our passwords, so head back to the Gemfile and tag it in:
```ruby
gem 'bcrypt', '~> 3.1.7'
```
_Note: the version number seen here is from running a `rails new` command with Rails 6.0.0... of course versions change and might be different for you if you're running the same command in the future... or in the past... or with different versions..._

Run `bundle install`.

This is not the blog that explains how to use `bcrypt`.  If you're unsure how it works, go find one or better yet, [read the docs].

Let's make some users.  Run

`rails g scaffold user name email password_digest`

followed by

`rails db:migrate`

To ensure we have hashed passwords and access to our `#password` setter, as well as the `#authenticate` method, let's add
`has_secure_password` to our `User` model:
```ruby
# inside app/models/user.rb
class User < ApplicationRecord
  has_secure_password
end
```

And let's create a couple users.  In `db/seeds.rb`:
```ruby
User.create(name: "Francine", email: "francine@francine.com", password: "francine")
User.create(name: "Franklin", email: "franklin@franklin.com", password: "franklin")
```

Run

`rails db:seed` followed by `rails c` so we can check our users exist:

```
// ♥ rails c
User.Running via Spring preloader in process 12138
couLoading development environment (Rails 6.0.0)
2.6.1 :001 > User.count
   (0.4ms)  SELECT sqlite_version(*)
   (0.4ms)  SELECT COUNT(*) FROM "users"
 => 2
2.6.1 :002 > User.first
  User Load (0.4ms)  SELECT "users".* FROM "users" ORDER BY "users"."id" ASC LIMIT ?  [["LIMIT", 1]]
 => #<User id: 1, name: "Francine", email: "francine@francine.com", password_digest: [FILTERED], created_at: "2019-09-30 13:16:30", updated_at: "2019-09-30 13:16:30">
```
This is looking good.

We have scaffolded a user, added `bcrypt`, and created some seed data.  Let's have a look at what Rails's scaffold generator gives us for a controller since we used the `--api` flag.  Notice we have a two private methods, `#set_user` to grab the user when the `:id` exists in the URL, and `#user_params` to enforce the strong params feature.  Notice the absence of `#new` and `#edit` actions, since we'll now be relying on JS for all HTML generation, including forms.  Finally, notice each controller action renders a response in JSON format, which is what we want.  Ideally, we ought to use a serializer to format and dictate exactly what to include in our JSON responses.  Options include gems like [Jbuilder] and [FastJsonapi].  We could also build our own custom serializer, build inline response hashes, or just use the [`#to_json`] method, which is what we'll do in this example for simplicity.

The `#index` action is quite simple here:
```ruby
# GET /users
def index
  @users = User.all

  render json: @users.to_json(only: [:id, :name, :email]), status: :ok
end
```

Let's run our Rails server and see what we get.  We've left the default settings, so our Rails server is running on `http://localhost:3000`.  _Another note here: it might be a good idea to namespace your API with "api" and/or a version number, such as `http://localhost:3000/api/v1/users`.  Again, not the focus of this blog, so we're sticking to bare bones on this front._  Navigating to `http://localhost:3000/users` gives us our JSON response:

```json
[
  {
    "id": 1,
    "name": "Francine",
    "email": "francine@francine.com"
  },
  {
    "id": 2,
    "name": "Franklin",
    "email": "franklin@franklin.com"
  }
]
```

OK let's see what we've got when our JS frontend talks to our Rails backend!  What frontend?  Your frontend might be a [React] application or plain old Vanilla JS.  For the sake of this blog, we'll be serving our JS frontend from `http://localhost:8000`.  Let's see what happens when we fire off a `fetch` request to get our users.

```javascript
  fetch("http://localhost:3000/users")
    .then(response => response.json())
    .then(console.log)
```

And we check our console... success!
```
(2) [{…}, {…}]
  0: {id: 1, name: "Francine", email: "francine@francine.com"}
  1: {id: 2, name: "Franklin", email: "franklin@franklin.com"}
  length: 2
  __proto__: Array(0)
```

Done.  Now we've got a JS frontend communicating with a Rails JSON API backend.  What did we come here for, again?  Oh yeah, auth.  We're not close to done.  OK, what is auth, anyway?  It's actually two things:

**Authentication**: Ensure the user is who the user claims to be.

**Authorization**: Once we've established the user's identity, ensure the user is allowed to do the thing the user is trying to do.

This blog focuses on authentication.  Although we briefly mentioned authorization in the quick guide.

To keep track of user sessions, we need signup, login, and logout functionality.  Let's start with signup.  Signup represents creating a user.  From there, it's up to you, the developer, whether you send the user back to a login page or log the user in automatically on signup.  Let's go with the latter for this example.

Suppose we have a signup form that gathers a user's name, email, and password.  We put the attributes into an object and pass it to the function we're using to send our POST request.  We might put together a request that looks something like this:
```javascript
  // suppose our `userData` is `{user: {name: "Felicia", email: "felicia@felicia.com", password: "felicia"}}`
  fetch("http://localhost:3000", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Accept": "application/json"
    },
    body: JSON.stringify(userData)
  })
    .then(response => response.json())
    .then(console.log)
```
_Note: notice the `:user` key in `userData`... Rails will attempt to [wrap params] in the appropriate key and it might work as long as all properties match columns in the database table.  Since bcrypt uses a column of `password_digest` but a setter method of `#password=`, we have to specify what to allow.  Here, we're choosing to wrap the incoming params under a top-level key of `:user`, while adjusting our strong params method as follows:_
```ruby
def user_params
  params.require(:user).permit(:name, :email, :password)
end
```
_Be careful using `scaffold` generators as they might not give you the code you need, or too much or too little._

Anyway, now let's see what the console shows us:
```
> {id: 5, name: "Felicia", email: "felicia@felicia.com"}
```
Success!  But wait, we're talking about auth...?  Right now, even though we can create a user, there is no session.  So neither the frontend nor the backend is aware of who the current user is.  Let's fix that.

Let's head back to our backend to continue our authentication configuration.  Hopefully you are familiar with using a basic auth setup on a full Rails app -- one where Rails is serving the views via `.erb` files.  Basically, we're going to use [Rails's `session` hash] to store an identifiable bit of info about a logged in user after signing up or logging in.  Logging out involves clearing the `session` hash.  Here are some likely helper methods we might include in `ApplicationController`:

```ruby
  def current_user
    User.find_by(id: session[:user_id])
  end

  def logged_in?
    !!current_user
  end
```

In our `users#create` action of a full Rails app, you may recall seeing code something like this:
```ruby
# POST /users
def create
  @user = User.new(user_params)
  if @user.save
    # the act of logging in is really just adding a key/value pair to the session hash
    session[:user_id] = @user.id
    redirect_to @user
  else
    flash[:message] = "Failed to create user"
    render :new
  end
end
```
Likewise, once we have users signed up, perhaps we use a `SessionsController` or `AuthController` to handle logging in and out, something like:

```ruby
# POST /login
def login
  @user = User.find_by(email: params[:user][:email])
  if @user && @user.authenticate(params[:user][:password])
    session[:user_id] = @user.id
    # then redirect to a landing page or whatever
  else
    # communicate to the user their credentials are invalid, maybe using a flash message
  end
end

# DELETE /logout
def logout
  session.clear
  # redirect back to a home or landing page
end
```

Since our controllers are now serving JSON, let's tweak `users#create`, `sessions#login` and `sessions#logout` accordingly:

```ruby
# UsersController, POST /users
def create
  @user = User.new(user_params)
  if @user.save
    # still just adding a key/value pair to the session hash
    session[:user_id] = @user.id
    render json: @user.to_json(only: [:id, :name, :email]), status: :created
  else
    render json: @user.errors, status: :unprocessable_entity
  end
end

# in SessionsController
def login
  @user = User.find_by(email: params[:user][:email])
  if @user && @user.authenticate(params[:user][:password])
    session[:user_id] = @user.id
    render json: @user.to_json(only: [:id, :name, :email]), status: :ok
  else
    render json: {
      error: "Invalid credentials"
    }, status: :unauthorized
  end
end

def logout
  session.clear
  render json: {
    message: "Successfully logged out"
  }, status: :ok
end
```

One more controller action that will probably prove useful is an action to get the current user if there is one.  This would likely be an endpoint we'll hit as soon as the app loads, so on `"DOMContentLoaded"` with JS or in `componentDidMount()` or `useEffect()`, if you're using [React].  Something like:

```ruby
# in `SessionsController` or `UsersController`, whichever you prefer
def get_current_user
  if logged_in?
    render json: current_user.to_json(only: [:id, :name, :email])
  else
    render json: {
      message: "No one is currently logged in"
    }
  end
end
```
So let's put this together with a new user and see what happens.  The test will be as follows:  we'll send a POST request to sign up a new user.  With the code we added to `users#create`, adding the user's id to the `session` hash, we should be logged in and the current user should appear in the console on a page refresh (remember that in our POST `/users` request shown earlier we just passed `console.log` as our callback to the second `.then()`).

But alas, what we're still seeing is:
```
the current user is {message: "No one is currently logged in"}
```

This is because the middleware Rails uses to include the `session` is removed due to the `--api` flag.  Here are the steps to get it back:

Head to `config/application.rb`.  Inside the `Application` class, add these two lines of code:

```ruby
config.middleware.use ActionDispatch::Cookies
config.middleware.use ActionDispatch::Session::CookieStore, key: '_cookie_name', expire_after: 7.days, httponly: true
```

We are bringing back middleware for cookies and sessions... who doesn't love a good session with some cookies?!  Options for [`CookieStore`] include:
- `:key` is just that -- the key the cookie is stored under in the browser.
- `:expire_after` allows us to set an expiration time for our cookies (the more sensitive the app's data, the shorter the cookie's lifespan should be).
- `:httponly` is an extra option to ensure our cookies are less vulnerable to an [XSS attack].
- `:secure` Defaults to `false`, but add `secure: true` only if running on HTTPS.  `localhost`, does not, so it's not included in the example code.

Now we have to explicitly tell Rails to use the middleware we just turned back on.  We can do this on a controller-specific basis, but the easiest way is to just add to `ApplicationController`:

```ruby
include ActionController::Cookies
```

Since all other controllers inherit from `ApplicationController`, they all get access to the tools we need now.  Since we've made configuration changes, restart your Rails server.  Now, all is well with the world.  Or, is it?

_Goes back, fills out sign up form, submits, annnddddd..._

```
the current user is {message: "No one is currently logged in"}
```

Oh, no!  Still not working!  You may have noticed something... we've 'turned on' Rails's `session` and cookies abilities, but so what?  Rails used to control the entire app, frontend and backend, so it was magically self-aware of what it included in its own requests and renderings and that it should include session cookies in its responses and look for them in incoming requests...  this may be an oversimplification, but the goal here is not to dive down the rabbit hole of cookies here.  Instead, let's be practical:  we need Rails to include session cookies in its requests, which we are doing now.  But we also need Rails to look for those cookies in _incoming_ AJAX requests now, too -- requests coming from external domains, i.e. from our browser.  We need to tell Rails two things: first, what domain(s) is(are) authorized to make AJAX requests, and second, to look for session cookies on those incoming requests.  Back to `config/initializers/cors.rb`:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    # 'whitelist' our origin, which for this example is `localhost:8000`
    origins 'http://localhost:8000'

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true
      # ^^ the `credentials: true` key/value pair tells Rails, "please look for incoming session cookie information!"
  end
end
```
Ok, we must be getting close... right?
_Restarts Rails server, fills out signup form with Darlene's info, refreshes to see if we're logged in, and ..._

```
the current user is {message: "No one is currently logged in"}
```

If you've been following along, your user database is probably growing in size, and your list of creative names to use is probably shrinking, hopefully not along with your patience.  ;p

What gives?  So not only did we need to tell Rails to look out for session cookie info, we also need to tell _JavaScript_ to _include_ said cookies in our AJAX requests!  BUT WAIT... Rails is, well, Ruby on Rails, with tons of helper methods and "magic", and JavaScript is, well, not Rails!  And we've disconnected our application so that we have a totally separate frontend and backend.  So how do we take this "final" step?  We Google, "how to include credentials in a fetch request"... And if we're me, we tack "MDN" to the front of that, so "MDN how to include credentials in a fetch request"

The first hit is [MDN's "Using Fetch"] article, where it says

```
By default, `fetch` won't send or receive any cookies from the server, resulting in unauthenticated requests if the site relies on maintaining a user session (to send cookies, the credentials `init option` must be set).
```
https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch

If we click ["init option"], we're taken to MDN's _[WindowOrWorkerGlobalScope.fetch()]_ article, where it says

```
init | Optional
  An options object containing any custom settings that you want to apply to the request. The possible options are:
```
  _...a whole bunch of options are described here here, including (hopefully) familiar properties such as `headers` and `method`, but `credentials` is what we want:_
```
  credentials: The request credentials you want to use for the request: omit, same-origin, or include. To automatically send cookies for the current domain, this option must be provided. Starting with Chrome 50, this property also takes a FederatedCredential instance or a PasswordCredential instance.
```
https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch#Parameters

To get a little more info, we might head to [MDN's "Request.credentials"] article, and we find out a little more about the `include` value of the `credentials` option:
```
include: Always send user credentials (cookies, basic http auth, etc..), even for cross-origin calls.
```
https://developer.mozilla.org/en-US/docs/Web/API/Request/credentials

BINGO!

So now, we need to add that option to our fetch requests, both for signing up and getting the current user:

```javascript
// suppose our `userData` is `{user: {name: "Mo", email: "mo@mo.com", password: "password"}}`
fetch("http://localhost:3000/users", {
  method: "POST",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    "Accept": "application/json"
  },
  body: JSON.stringify(userData)
})
  .then(response => response.json())
  .then(console.log)
```
and
```js
fetch("http://localhost:3000/get_current_user", {
  method: "GET",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    "Accept": "application/json"
  }
})
  .then(response => response.json())
  .then(console.log)
```

And now, when we go to sign Mo up, then refresh the page and check the console:

```
the current user is {id: 10, name: "Mo", email: "mo@mo.com"}
```

YASSSSSS!!!!  Now, of course, instead of just logging to the console, we'd likely do something with our user on the front end.  Maybe stash it into whatever state management system we're using, whether it's React state, Redux, or any other flavor or JS we like.  And, of course, since we're building an SPA, instead of refreshing the page to get results, we'll want to update our DOM by invoking the proper methods and functions we've built to do so on our front end.  Which is a whole other thing, altogether.

What do you think?  That's pretty much it!  Feel free to leave comments below.

One more thing -- here are a couple common config errors you might encounter...

If you leave the wildcard `'*'` as your origin and add `credentials: true`, Rails won't even allow the server to run:

```
Allowing credentials for wildcard origins is insecure. Please specify more restrictive origins or set 'credentials' to false in your CORS configuration. (Rack::Cors::Resource::CorsMisconfigurationError)
```

Adding `credentials: "include"` to your AJAX requests without properly whitelisting your origin(s) will trigger a CORS error when the request is made:
```
Access to fetch at 'http://localhost:3000/get_current_user' from origin 'http://localhost:8000' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '*' when the request's credentials mode is 'include'.
```
Whitelisting origin(s), adding `credentials: "include"` to AJAX requests, but failing to add `credentials: true` in your Rails CORS config results in a slightly different error:

```
Access to fetch at 'http://localhost:3000/users' from origin 'http://localhost:8000' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: The value of the 'Access-Control-Allow-Credentials' header in the response is '' which must be 'true' when the request's credentials mode is 'include'.
```


[Rails JSON API]: https://guides.rubyonrails.org/api_app.html
[React]: https://reactjs.org
[Jbuilder]: https://github.com/rails/jbuilder
[`#to_json`]: https://apidock.com/rails/ActiveRecord/Serialization/to_json
[Cross Origin Resource Sharing]: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
[Aysychronous JavaScript and XML]: https://developer.mozilla.org/en-US/docs/Glossary/AJAX
[jQuery's `.ajax()`]: https://api.jquery.com/jquery.ajax/
[read the docs]: https://github.com/codahale/bcrypt-ruby
[wrap params]: https://api.rubyonrails.org/v5.2.3/classes/ActionController/ParamsWrapper.html
[Fetch API]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
[FastJsonapi]: https://github.com/Netflix/fast_jsonapi
[`CookieStore`]: https://api.rubyonrails.org/v5.2.1/classes/ActionDispatch/Session/CookieStore.html
[MDN's "Using Fetch"]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
["init option"]: https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch#Parameters
[WindowOrWorkerGlobalScope.fetch()]: https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch#Parameters
[HTTPS]: https://www.globalsign.com/en/blog/the-difference-between-http-and-https/
[MDN's "Request.credentials"]: https://developer.mozilla.org/en-US/docs/Web/API/Request/credentials
[XSS attack]: https://humanwhocodes.com/blog/2009/05/12/cookies-and-security/
[Rails's `session` hash]: https://guides.rubyonrails.org/security.html#sessions
