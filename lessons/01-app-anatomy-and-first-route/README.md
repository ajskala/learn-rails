# Lesson 1 — App Anatomy, MVC, and Your First Route

## The single biggest mindset shift from Node

Node/Express is explicit: you `import`/`require` everything, you wire up every
route by hand in code, nothing happens unless you wrote the line that makes it
happen. Rails' core philosophy is the opposite: **convention over configuration**.
If you name and place a file the way Rails expects, Rails finds and wires it up
*automatically* — no `import`, no manual registration. This feels like magic at
first and is the #1 source of "why isn't this working" confusion for newcomers —
almost always the fix is "the filename or class name doesn't match what Rails
expects." We'll hit this directly in the exercise.

## The folder structure (MVC)

Open `app/` inside `learn-rails/app` (the generated Rails app) and look at:

```
app/
  controllers/   # handle incoming requests, decide what to do, what to render
  models/        # data + business logic (backed by the database — Lesson 2)
  views/         # templates that render HTML (ERB — embedded Ruby)
config/
  routes.rb      # maps URLs + HTTP verbs -> controller actions
db/
  schema.rb      # current database structure (Lesson 2)
test/            # minitest test suite — this is our "tsc"/"mypy" equivalent:
                 # run `bin/rails test` to check your work
```

**Request flow:** browser hits a URL → `config/routes.rb` matches it to a
controller + action → the controller action runs (Ruby method) → it renders a
view (or redirects, or returns JSON) → HTML goes back to the browser.

## Routes

`config/routes.rb` maps an HTTP verb + path to `controller#action`:

```ruby
Rails.application.routes.draw do
  get "hello", to: "pages#hello"
  root "pages#hello"   # what GET / renders
end
```

`"pages#hello"` means: `PagesController`, `hello` action (method).

## Controllers

```ruby
# app/controllers/pages_controller.rb
class PagesController < ApplicationController
  def hello
    @message = "Hello from Rails"
  end
end
```

Naming convention (Zeitwerk autoloading — this is the "magic" part): the file
`app/controllers/pages_controller.rb` must define a class named exactly
`PagesController`. Get the casing/naming wrong and Rails won't find it — no
`import` statement will save you, because there isn't one. `snake_case` filename ↔
`PascalCase` class name, always.

Every public method on a controller that matches a route is an **action**. An
action doesn't need an explicit `return` — instance variables (`@message`, note
the `@`) you set here are automatically visible inside the matching view. That's
implicit data-passing you'd do explicitly in Express (e.g. `res.render("hello", {
message })`).

## Views (ERB)

```erb
<%# app/views/pages/hello.html.erb %>
<h1><%= @message %></h1>
```

- `<%= ... %>` evaluates Ruby and **inserts** the result into the HTML (like TS
  template literal `${...}`).
- `<% ... %>` evaluates Ruby but inserts **nothing** — used for control flow
  (`if`, `each` loops) around HTML.

The view path is also convention: action `hello` on `PagesController` looks for
`app/views/pages/hello.html.erb` by default — `pages` from the controller name
(minus `Controller`, snake_cased), `hello` from the action name.

## Ruby syntax notes as you'll need them

```ruby
def greet(name)         # no type annotations required (Ruby is dynamic,
  "Hi, #{name}"         # like plain Python) — string interpolation is #{...}
end                      # inside double-quoted strings, not single-quoted

class Foo
  def bar                # method definitions: def ... end, no braces
    42
  end
end                       # every def, class, module, if, etc. needs a matching `end`

:symbol                   # a Symbol — an immutable, cheap identifier/label,
                           # used constantly for hash keys and options, e.g.
                           # `to: "pages#hello"` above — that's a Symbol key
```

## Checking your work

```
cd ~/workspace/learn-rails/app
bin/rails test              # run the test suite (our "tsc"/"mypy" check step)
bin/rails server             # start the dev server at http://localhost:3000
```

## Exercise

1. Read through `app/config/routes.rb` and the existing `app/controllers/` and
   `app/views/` directories (there's an `ApplicationController` and
   `layouts/application.html.erb` already — don't worry about those yet).

2. Add a route: `GET /hello` → `pages#hello`, and make it the app's `root` route
   too (`root "pages#hello"`).

3. Create `app/controllers/pages_controller.rb` by hand (don't use the generator
   yet) with a `hello` action that sets `@message` to any string you like.

4. Create the view `app/views/pages/hello.html.erb` by hand, with an `<h1>` that
   outputs `@message`.

5. Verify it two ways:
   - `bin/rails server`, then in another terminal:
     `curl http://localhost:3000/hello` — should show your `<h1>` with your
     message. Stop the server after with Ctrl-C.
   - Write a test in `test/controllers/pages_controller_test.rb` (create the file)
     asserting the response is successful and contains your message:

     ```ruby
     require "test_helper"

     class PagesControllerTest < ActionDispatch::IntegrationTest
       test "hello renders the message" do
         get hello_path
         assert_response :success
         assert_select "h1", "Hello from Rails" # match whatever you actually set
       end
     end
     ```

     Run `bin/rails test` and get it green.

6. (Try breaking it on purpose, then fixing it — this is the important part for
   building intuition): rename the class in `pages_controller.rb` to something
   that *doesn't* match the filename (e.g. `class PageController` — singular).
   Run `bin/rails server` and hit the route again. Read the error Rails gives
   you. Then rename it back to `PagesController` and confirm it works again.
   This is the convention-over-configuration failure mode you'll hit most often
   in real Rails work — good to see it once deliberately.
