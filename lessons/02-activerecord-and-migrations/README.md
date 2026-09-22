# Lesson 2 — ActiveRecord, Migrations, and Your First Real Data

## Migrations — versioned changes to the database schema

Rails never has you write SQL `CREATE TABLE` by hand for day-to-day work.
Instead, you write a **migration**: a Ruby file describing one change to the
schema, which Rails can apply (`db:migrate`) or roll back (`db:rollback`). Every
migration that's ever been run is tracked in a `schema_migrations` table, so
Rails always knows exactly which changes have been applied.

Generate one with the `rails generate` (`rails g`) command — this is the code
generator you deliberately avoided in Lesson 1 to learn the manual mechanics
first. Now that you know what it produces, it's a legitimate productivity tool:

```
bin/rails generate migration CreatePosts title:string body:text
```

This creates a timestamped file under `db/migrate/`, something like:

```ruby
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body

      t.timestamps
    end
  end
end
```

`t.timestamps` is a Rails convention: it adds `created_at`/`updated_at` columns
that ActiveRecord maintains automatically on every save — you never set these
yourself. Table names are plural (`posts`), snake_case — this maps to a model
class named `Post` (singular, PascalCase). This naming convention is exactly the
same category of "magic" as Lesson 1's controller autoloading: get the
plural/singular pairing wrong and ActiveRecord won't find the table.

Apply it:

```
bin/rails db:migrate
```

This updates `db/schema.rb` (the current, human-readable snapshot of your whole
schema — never hand-edit this file) and creates the `posts` table in
`storage/development.sqlite3`.

## Models — ActiveRecord

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true
end
```

That's the entire model — no need to declare `title`/`body` as attributes
anywhere in Ruby code. `ActiveRecord::Base` (via `ApplicationRecord`) inspects
the actual database table's columns at load time and generates matching
attribute methods automatically. This is convention over configuration again,
one level deeper than controllers: the **database schema itself** is the source
of truth for what attributes a model has.

`validates :title, presence: true` is a model-level validation — it runs before
`save`/`create`, and a save that violates it fails (returns `false` /
`save!` raises) without hitting the database at all. Validations are Ruby-level
business rules, separate from any database-level constraints.

## Exploring in the Rails console

Before writing any controller code, get comfortable with the model directly:

```
bin/rails console
```

```ruby
Post.create(title: "Hello Rails", body: "My first post")
Post.create(body: "No title, this should fail validation")   # returns false — no title
Post.all
Post.count
Post.first.title
Post.where(title: "Hello Rails")
exit
```

This is a real Ruby REPL with your whole app loaded — the fastest way to poke at
models while learning, no server or browser needed.

## RESTful routes with `resources`

Lesson 1 had you write `get "hello", to: "pages#hello"` by hand — one route, one
action. Rails apps mostly deal in a repeating pattern instead: list all
records, show one, create, edit, update, delete. `resources` generates all of
that in one line:

```ruby
resources :posts, only: [:index, :show]
```

Run `bin/rails routes | grep posts` after adding this to see exactly what it
expanded into — two routes, `GET /posts` → `posts#index` and `GET /posts/:id` →
`posts#show`. (`resources :posts` with no `only:` would generate all seven
conventional REST actions; we're deliberately scoping to two for this lesson.)

## Controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end

  def show
    @post = Post.find(params[:id])
  end
end
```

`params[:id]` reads the `:id` segment from the URL (`/posts/3` → `params[:id] ==
"3"`) — `resources` is what wires that URL segment to `:id` automatically.
`Post.find` raises `ActiveRecord::RecordNotFound` (Rails turns this into a 404)
if no row matches — contrast with `Post.where(id: ...)`, which returns an empty
result instead of raising.

## Views — looping and linking

```erb
<%# app/views/posts/index.html.erb %>
<h1>Posts</h1>
<ul>
  <% @posts.each do |post| %>
    <li><%= link_to post.title, post %></li>
  <% end %>
</ul>
```

```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>
```

`link_to post.title, post` generates an `<a>` tag — first argument is the link
text, second is where it points. Passing the `post` object itself (not a
hand-built URL string) works because Rails knows how to turn an ActiveRecord
object into its `show` URL, again via the `resources` routing convention.

## Exercise

1. Generate and run the `CreatePosts` migration (`title:string`, `body:text`) as
   shown above.
2. Create `app/models/post.rb` with the `title` presence validation.
3. Use `bin/rails console` to create at least 2 valid posts and confirm the
   invalid case (no title) actually fails — this step is exploratory, nothing to
   write into a file for it.
4. Add `resources :posts, only: [:index, :show]` to `config/routes.rb`. Run
   `bin/rails routes | grep posts` and confirm you see both routes.
5. Create `PostsController` with `index` and `show` actions as shown above.
6. Create `app/views/posts/index.html.erb` and `app/views/posts/show.html.erb`.
7. Verify: `bin/rails server`, then visit `http://localhost:3000/posts` in a
   browser (or `curl`) — you should see your posts listed, and clicking one (or
   curling `/posts/1`) should show its title and body.
8. Write a test in `test/models/post_test.rb`:

   ```ruby
   require "test_helper"

   class PostTest < ActiveSupport::TestCase
     test "requires a title" do
       post = Post.new(body: "no title here")
       assert_not post.save
     end

     test "saves with a title" do
       post = Post.new(title: "Valid", body: "has a title")
       assert post.save
     end
   end
   ```

   And a request test in `test/controllers/posts_controller_test.rb`:

   ```ruby
   require "test_helper"

   class PostsControllerTest < ActionDispatch::IntegrationTest
     test "index lists posts" do
       Post.create!(title: "Test Post", body: "content")
       get posts_path
       assert_response :success
       assert_select "li", text: /Test Post/
     end
   end
   ```

   Run `bin/rails test` and get both green. (Note these tests create their own
   data — Rails wraps each test in a transaction that's rolled back afterward, so
   they don't depend on or pollute whatever you created in the console earlier.)
