# learn-rails

Structured lessons, same idea as `learn-ts-node` and `learn-python`, adapted to how
Rails actually works: Rails is a full-stack MVC framework, not a scripting language,
so instead of one isolated `exercise.py`/`.ts` file per lesson, there's **one shared
Rails app** (`app/`) that every lesson builds on incrementally. Each lesson's
`README.md` under `lessons/` explains concepts and gives you TODOs to implement
directly inside `app/`.

## Setup (done)

- Ruby 3.4.10 installed via `rbenv` (system Ruby was 2.6.10 — years out of date,
  incompatible with modern Rails), pinned for this project via `.ruby-version`.
- Rails 8.1.3.1 installed as a gem.
- The app itself lives in `app/` (generated with `rails new`, SQLite + minitest,
  Rails' defaults — good for learning, nothing exotic).

## Workflow for every lesson

```
cd ~/workspace/learn-rails/app
eval "$(rbenv init -)"   # once per new terminal, so `ruby`/`rails`/`bundle` resolve
                          # to the rbenv-managed 3.4.10 instead of system Ruby
bin/rails test            # run the test suite — our "tsc"/"mypy" check step
bin/rails server           # start the dev server at http://localhost:3000
```

Unlike the TS/Python projects, there's no separate `exercise.ts`/`.py` per lesson —
Rails only really makes sense as one running app, so every lesson's TODOs get
implemented directly inside `app/`, building on the previous lesson's work.
