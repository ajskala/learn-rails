# Lesson 4 — Model Associations: `has_many` / `belongs_to`

Every lesson so far has had exactly one model. Real apps are graphs of related
models — this lesson adds `Comment`s that belong to a `Post`, the most common
relationship shape in Rails: one-to-many.

## The `rails generate model` generator

Lesson 2 used `rails generate migration` (migration file only, you wrote the
model by hand) so you'd learn the raw mechanics. `rails generate model` is one
level up — it creates the migration **and** a starter model file together, which
is the more common way to add a genuinely new model in practice:

```
bin/rails generate model Comment post:references body:text
```

`post:references` is a special column type — it doesn't just add a `post_id`
integer column, it also generates the matching `belongs_to :post` line directly
in the created `app/models/comment.rb`, and adds a **foreign key constraint** at
the database level (`foreign_key: true` in the migration) so the database itself
refuses to store a `comment` pointing at a `post_id` that doesn't exist — a
correctness guarantee below the Ruby layer entirely.

Check the generated migration and model before running it — you'll see:

```ruby
# db/migrate/..._create_comments.rb
create_table :comments do |t|
  t.references :post, null: false, foreign_key: true
  t.text :body
  t.timestamps
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
end
```

The generator also creates `test/models/comment_test.rb` (an empty placeholder,
harmless) and `test/fixtures/comments.yml` — this one is **not** harmless: it
comes pre-filled with two rows referencing `post: one` / `post: two`, fixture
labels that don't exist anywhere since we haven't created a `posts.yml`
fixture in this lesson (our tests build records directly with `create!`
instead). Rails loads *every* fixture file before *every* test runs, regardless
of which test file you're running — so this broken reference will fail with a
`Foreign key violations found in your fixture data` error the moment you run
`bin/rails test`, on a test that has nothing to do with comments. Fix it by
emptying `test/fixtures/comments.yml` back down to just its header comment.
This isn't a mistake you made — it's what the generator produces by default
whenever you don't also set up a matching fixture for the referenced
association, worth knowing since you'll hit this shape of error again in real
projects.

Run it: `bin/rails db:migrate`.

## The other half: `has_many` on `Post`

`belongs_to :post` was generated for you; `has_many` is not (Rails has no way to
know you want it — go add it to `app/models/post.rb` yourself):

```ruby
class Post < ApplicationRecord
  validates :title, presence: true
  has_many :comments, dependent: :destroy
end
```

`dependent: :destroy` is important and easy to forget: without it, deleting a
`Post` would leave its `comments` rows behind, pointing at a `post_id` that no
longer exists — orphaned data. With it, destroying a post first destroys all its
comments. (The foreign key constraint from `references` would actually *reject*
deleting a post with existing comments at the database level without this — try
it and see, per the exercise below.)

## What `has_many`/`belongs_to` actually buys you

Once both sides are declared, ActiveRecord gives you a small API for navigating
the relationship, in the Rails console:

```ruby
post = Post.first
post.comments                              # all comments belonging to this post
post.comments.create(body: "Nice post!")    # creates a comment, post_id set automatically
post.comments.count

comment = Comment.first
comment.post                                 # the Post this comment belongs to
comment.post.title
```

`post.comments.create(...)` is the idiomatic way to create an associated record
— it sets `post_id` for you, you never set foreign keys by hand.

## Nested routes

A comment only makes sense in the context of a post, so its routes nest under
`posts`:

```ruby
resources :posts do
  resources :comments, only: [:create, :destroy]
end
```

Run `bin/rails routes | grep comments` — you'll see paths like `POST
/posts/:post_id/comments` and `DELETE /posts/:post_id/comments/:id`. Note the
param name: `:post_id`, not `:id`, for the outer resource — you need both IDs to
identify a specific nested comment.

## `CommentsController`

```ruby
class CommentsController < ApplicationController
  def create
    @post = Post.find(params[:post_id])
    @post.comments.create(comment_params)
    redirect_to @post
  end

  def destroy
    @post = Post.find(params[:post_id])
    @post.comments.find(params[:id]).destroy
    redirect_to @post
  end

  private

  def comment_params
    params.require(:comment).permit(:body)
  end
end
```

Note `@post.comments.find(...)`, not `Comment.find(...)`, in `destroy` — scoping
the lookup through `@post.comments` means someone can't delete a comment
belonging to a *different* post just by guessing its ID in the URL. This is a
real, common security consideration in nested resources, not a style
preference.

## Showing and adding comments on the post page

```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>

<h2>Comments</h2>
<ul>
  <% @post.comments.each do |comment| %>
    <li>
      <%= comment.body %>
      <%= button_to "Delete", [@post, comment], method: :delete %>
    </li>
  <% end %>
</ul>

<%= form_with model: [@post, Comment.new] do |f| %>
  <%= f.text_area :body %>
  <%= f.submit "Add Comment" %>
<% end %>
```

`form_with model: [@post, Comment.new]` — an array tells Rails to build a
**nested** route (`/posts/:post_id/comments`) instead of a plain one; this is
the array form of the same trick `form_with model: post` used in Lesson 3.
Same idea for `button_to "Delete", [@post, comment], ...`.

## Exercise

1. Generate the `Comment` model as shown (`rails g model Comment post:references
   body:text`), inspect the generated migration and model, then run
   `bin/rails db:migrate`. Empty out `test/fixtures/comments.yml` down to just
   its header comment, per the note above.
2. Add `has_many :comments` (no `dependent:` option yet) to `Post`.
3. In `bin/rails console`: create a post, add two comments to it via
   `post.comments.create(...)`, confirm `post.comments.count` and
   `comment.post.title` both work.
4. Still in the console, call `post.destroy` on a post that has comments and
   watch it raise `ActiveRecord::InvalidForeignKey` — the database-level
   constraint from `references` rejecting the delete. Then add `dependent:
   :destroy` to the `has_many :comments` line in `Post`, restart the console
   (model changes aren't picked up by an already-running console), and confirm
   `post.destroy` now succeeds and also removes its comments.
5. Add the nested `resources :comments, only: [:create, :destroy]` under
   `resources :posts` in `routes.rb`. Confirm with `bin/rails routes | grep
   comments`.
6. Create `CommentsController` with `create`/`destroy` as shown, scoping lookups
   through `@post.comments`.
7. Update `app/views/posts/show.html.erb` to list comments and include the
   nested `form_with` for adding one.
8. Verify manually: `bin/rails server`, visit a post, add a comment through the
   form, delete it.
9. Add a request test to a new `test/controllers/comments_controller_test.rb`
   covering create and destroy, plus a model test in
   `test/models/post_test.rb` asserting that destroying a post also destroys its
   comments (create a post with a comment, call `post.destroy`, assert
   `Comment.exists?(comment.id)` is now false). Get `bin/rails test` green.
