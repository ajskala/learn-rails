# Lesson 3 — Forms and the Rest of CRUD

Lesson 2 gave `Post` two of the four CRUD operations: **R**ead (`index`,
`show`). This lesson adds **C**reate, **U**pdate, and **D**elete — the full set
`resources` is named after.

> **An environment gotcha already fixed for you**: this is the first lesson
> where a request needs a full session-cookie round trip (the CSRF token that
> `form_with` embeds is verified against your session). While building this
> lesson, that round trip hit `ArgumentError: wrong number of arguments (given
> 2, expected 1)` deep inside `ActiveSupport::JSON.decode` — a real
> incompatibility between `json` gem 3.0.2 (what `bundle install` resolved to)
> and this Rails version's session/cookie decryption code. Nothing you wrote
> would have caused or fixed this — it's already pinned around in the
> `Gemfile` (`gem "json", "~> 2.9"`). Mentioned here so that line in the
> `Gemfile` isn't a mystery, and so you know what a broken session actually
> looks like if you ever hit something similar in a real project: a bizarre
> `ArgumentError` from deep inside a dependency, on the very first request
> that touches a form.

## Expanding the route

```ruby
resources :posts
```

No `only:` — this generates all seven conventional REST routes. Run `bin/rails
routes | grep posts` after the change and compare against Lesson 2's two routes;
you'll see `new`/`create`/`edit`/`update`/`destroy` added, each mapped to a
specific HTTP verb (`GET /posts/new`, `POST /posts`, `GET /posts/:id/edit`,
`PATCH/PUT /posts/:id`, `DELETE /posts/:id`) — REST maps CRUD operations to HTTP
verbs, not just URL paths.

## Strong parameters — mass-assignment protection

```ruby
def post_params
  params.require(:post).permit(:title, :body)
end
```

`params` contains *everything* the client sent — you never want to blindly do
`Post.create(params[:post])` in a real app, because a client could send extra
fields you never intended to be settable (imagine a hypothetical `is_admin`
column). `.require(:post)` raises unless a `post` key is present; `.permit(:title,
:body)` allow-lists exactly which fields may pass through. This is a real
security control, not boilerplate — Rails calls the general problem "mass
assignment," and strong parameters is Rails' built-in answer to it.

## `new` and `create`

```ruby
def new
  @post = Post.new
end

def create
  @post = Post.new(post_params)
  if @post.save
    redirect_to @post, notice: "Post created."
  else
    render :new, status: :unprocessable_entity
  end
end
```

`new` builds an unsaved, blank `Post` for the form to bind to. `create` builds
one from submitted params and tries to save it: on success, `redirect_to @post`
sends a fresh `GET` to the new post's `show` page (redirect-after-post — this
avoids the classic "resubmit form on refresh" browser problem); on failure,
`render :new` re-renders the *same* form, now with `@post` carrying validation
errors, without redirecting (so the URL still reflects the failed attempt and
the browser's refresh button would just resubmit the same failing data —
sometimes acceptable for a learning exercise, avoided in production apps with
more care).

`redirect_to @post, notice: "..."` sets a one-time **flash** message, available
in the very next request only, then gone — perfect for "your action succeeded"
banners.

## The form itself — `form_with`

```erb
<%# app/views/posts/_form.html.erb — note the leading underscore: a PARTIAL %>
<%= form_with model: post do |f| %>
  <% if post.errors.any? %>
    <ul>
      <% post.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  <% end %>

  <%= f.label :title %>
  <%= f.text_field :title %>

  <%= f.label :body %>
  <%= f.text_area :body %>

  <%= f.submit %>
<% end %>
```

A filename starting with `_` is a **partial** — a reusable view fragment,
rendered into another view rather than as a full response on its own. The same
form works for both `new` and `edit`: `form_with model: post` inspects whether
`post` is a new, unsaved record or an existing one, and automatically points the
form at the right URL with the right HTTP verb (`POST /posts` vs `PATCH
/posts/:id`) — you never write that logic yourself. It also embeds the CSRF
authenticity token automatically (the same token you saw in the raw HTML back in
Lesson 2's `curl` output) — Rails' default cross-site-request-forgery
protection, on by default for every non-GET form.

`new.html.erb` and `edit.html.erb` both just render the shared partial:

```erb
<%# app/views/posts/new.html.erb %>
<h1>New Post</h1>
<%= render "form", post: @post %>
```

```erb
<%# app/views/posts/edit.html.erb %>
<h1>Edit Post</h1>
<%= render "form", post: @post %>
```

## `edit`, `update`, and `destroy`

```ruby
def edit
  @post = Post.find(params[:id])
end

def update
  @post = Post.find(params[:id])
  if @post.update(post_params)
    redirect_to @post, notice: "Post updated."
  else
    render :edit, status: :unprocessable_entity
  end
end

def destroy
  @post = Post.find(params[:id])
  @post.destroy
  redirect_to posts_path, notice: "Post deleted."
end
```

Same success/failure pattern as `create`. `destroy` has no failure branch to
worry about for a plain model like this — deleting doesn't run validations.

## Deleting from a view — `button_to`

A plain `<a>` link always sends `GET`, but `DELETE /posts/:id` needs the `DELETE`
verb. `button_to` renders a tiny auto-submitting form instead of a link, so it
can use any HTTP verb (and carries the CSRF token, same as `form_with`):

```erb
<%= button_to "Delete", post, method: :delete %>
```

## Flash messages in the layout

`notice:`/`alert:` passed to `redirect_to` only actually appears if something
renders it. Add this once to the shared layout so every page picks it up:

```erb
<%# app/views/layouts/application.html.erb, inside <body> %>
<% if flash[:notice] %>
  <p><%= flash[:notice] %></p>
<% end %>
```

## Exercise

1. Change `resources :posts, only: [:index, :show]` to plain `resources :posts`.
   Confirm the new routes with `bin/rails routes | grep posts`.
2. Add a private `post_params` method to `PostsController` using strong
   parameters (`:title`, `:body`).
3. Add `new` and `create` actions.
4. Create `app/views/posts/_form.html.erb` (the shared partial, with validation
   error display) and `app/views/posts/new.html.erb`.
5. Add `edit` and `update` actions, and `app/views/posts/edit.html.erb`.
6. Add a `destroy` action. Add a delete `button_to` to `show.html.erb` (or
   `index.html.erb`, your choice).
7. Add the flash message snippet to the layout.
8. Verify manually: `bin/rails server`, then in a browser, create a post via the
   form, confirm the flash message shows, edit it, delete it. Also try submitting
   the new-post form with a blank title and confirm the validation error shows
   without crashing.
9. Add request tests to `test/controllers/posts_controller_test.rb` for create,
   update, and destroy (one each, following the pattern from Lesson 2's `index`
   test). Get `bin/rails test` green.
