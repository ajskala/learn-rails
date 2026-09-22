# learn-rails

Structured, self-checking lessons for learning Ruby on Rails, written from the
perspective of someone who already knows TypeScript/Node. Rails is a full-stack
MVC framework, not a scripting language, so instead of one isolated exercise file
per lesson like the companion `learn-ts-node`/`learn-python` repos, there's **one
shared Rails app** (`app/`) that every lesson builds on incrementally. Each
lesson's `README.md` under `lessons/` explains concepts and gives you TODOs to
implement directly inside `app/`.

## Prerequisites

- **A Ruby version manager** — required. The system/OS-bundled Ruby on most
  machines (especially macOS) is old and unsuitable for modern Rails.
  [rbenv](https://github.com/rbenv/rbenv) is what this project was built with;
  [asdf](https://asdf-vm.com/) or [RVM](https://rvm.io/) work too. On macOS:
  ```
  brew install rbenv ruby-build
  ```
- **Ruby 3.2+** (built and verified on 3.4.10 — Rails 8 requires at least 3.2).
  Install it through your version manager, e.g.:
  ```
  rbenv install 3.4.10
  ```
- **Bundler** (installs with Ruby/Rails, no separate step needed).
- **SQLite3** — this app uses SQLite (Rails' default), which ships preinstalled
  on macOS and most Linux distros. If `bundle install` complains about a missing
  `sqlite3` library, install it via your OS package manager first (e.g.
  `brew install sqlite3` on macOS, `apt install libsqlite3-dev` on Debian/Ubuntu).

## Setup

```
git clone https://github.com/ajskala/learn-rails.git
cd learn-rails

eval "$(rbenv init -)"    # so `ruby`/`gem`/`bundle` resolve to the rbenv-managed
                            # Ruby instead of your system Ruby — do this once per
                            # new terminal, every time you work in this repo
rbenv install -s $(cat .ruby-version)   # installs the pinned Ruby version if you
                                          # don't already have it

gem install rails           # installs Rails itself as a gem, if not already present
rbenv rehash

cd app
bundle install               # installs this app's gems (Rails, etc.) from Gemfile.lock
bin/rails db:prepare          # creates and migrates the SQLite database

bin/rails test                 # should report 0 failures — confirms your setup works
```

**On `config/master.key`**: this repo deliberately does *not* include
`app/config/master.key` — it's Rails' decryption key for
`config/credentials.yml.enc` and must never be committed (it's gitignored by
default). None of the current lessons need it — the app boots fine in
development/test without it, verified above. If a later lesson touches Rails
credentials and you see a `MissingKeyError`, generate your own local key with
`bin/rails credentials:edit` from inside `app/`.

## Workflow for every lesson

```
cd ~/workspace/learn-rails/app   # or wherever you cloned it
eval "$(rbenv init -)"            # once per new terminal
bin/rails test                     # run the test suite — the "tsc"/"mypy" check step
bin/rails server                    # start the dev server at http://localhost:3000
```

Read a lesson's `README.md` under `lessons/`, then implement its TODOs directly in
`app/` (routes, controllers, views, models — whatever the lesson calls for).
Verify with `bin/rails test` and/or by hitting the running server.

## Lessons

1. **`01-app-anatomy-and-first-route`** — MVC request flow, Rails'
   convention-over-configuration philosophy, routes, controllers, ERB views, and
   a minitest-based check step.
