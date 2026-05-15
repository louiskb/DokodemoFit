# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Dokodemo Fit** — a Rails 7.1 AI app that generates personalised workout routines and meal plans from a user's stated goals and available equipment. The AI returns Bootstrap-styled HTML directly (no Markdown rendering layer), which is rendered inside the routine's show page.

Built at Le Wagon Tokyo by Séamus, Louis and Youssef. Not actively maintained — kept hosted for portfolio purposes.

## Stack

- Ruby 3.3.5 / Rails 7.1.5
- PostgreSQL (`pg`)
- Puma; default Action Cable adapter (no Solid Cable here, unlike sibling project Market Sensei)
- Devise for auth
- Bootstrap 5.3 via `bootstrap` gem + `sassc-rails`; Hotwire (Turbo + Stimulus) over Importmap
- `simple_form` (heartcombo/simple_form from GitHub) — use `simple_form_for` / `f.input` / `f.button :submit`, never plain `form_with`
- `ruby_llm` via the SuzukiRyuichiro fork (temporary workaround for upstream bug) — talks to GitHub Models endpoint (`https://models.inference.ai.azure.com`) using `ENV["GITHUB_TOKEN"]`. GitHub Models has a hard token-rate limit, so the app may fail under any real load (this is noted in the README as a known limitation).
- Heroku deploy via `Procfile` (`release: rails db:migrate` only — runs migrations on each release; `web` falls back to Heroku's Puma default)

## Commands

```bash
bin/setup                          # install deps + prep db
bin/rails server                   # dev server (port 3000)
bin/rails console                  # Rails console

bin/rails db:create db:migrate     # set up local DB
bin/rails db:seed                  # seed exercises

bin/rails test                     # full Minitest suite
bin/rails test test/path/to_test.rb        # single file
bin/rails test test/models/routine_test.rb -n test_method_name   # single test

bundle exec rubocop                # lint (see .rubocop.yml — Layout/LineLength max 120, many cops disabled)
```

## Required env vars

- `GITHUB_TOKEN` — GitHub Models API token, used as `openai_api_key` in `config/initializers/ruby_llm.rb`
- A `.env` file exists locally for development (loaded via `dotenv-rails`)

## Architecture

**One concept dominates: `Routine` is the chat.** A `User has_many :routines`. `Routine` plays the role that `Chat` plays in most ruby_llm apps — `Message belongs_to :routine`, and `Message#chat` is aliased to `routine` (see `app/models/message.rb`) so that `ruby_llm`'s `acts_as_message` plumbing works against routines instead of a separate chats table.

A `Routine has_many :routine_exercises, has_many :exercises, through: :routine_exercises` — the exercise library is a seedable join, but the actual workout plan is the AI-generated HTML stored in the routine's latest assistant message.

**Two services orchestrate the AI:**
- `AiInitialPromptService` (called from `RoutinesController#create`) — uses `Message#ai_initial_prompt`, which is a large prompt that includes the routine's `goal`, `equipment`, `comments` and an inline HTML/Bootstrap template the AI is asked to mimic. The AI returns raw HTML which is then rendered with `raw` / `sanitize` on the show page.
- `UpdatePromptService` (called from `RoutinesController#update`) — uses `Message#update_prompt`, which embeds the previous assistant message (the HTML plan) and the latest user message (the feedback) and asks for an updated HTML plan.

**Important quirk:** prompts return *HTML*, not Markdown. There is no Markdown parser in the Gemfile. If you change the prompts, keep the HTML-only contract — downstream views assume it.

**Known bug to be aware of:** in `RoutinesController#show / #edit / #destroy` the access check uses `=` instead of `==` (`unless @routine.user = current_user`) — this is an assignment, not a comparison, and will silently overwrite ownership. Fix before adding any new authorization-sensitive code on top of it.

## Conventions

- Double quotes everywhere (matches Louis's global preference; `.rubocop.yml` disables `Style/StringLiterals` so it's not enforced — write doubles anyway)
- `ENV.fetch("X", nil)` over `ENV["X"]` in application code — match existing style within a file unless refactoring it
- `app/services/` holds AI orchestration; controllers stay thin
- HTML responses from the AI are embedded directly into views — don't add a Markdown layer unless you also update the prompts

## Heroku deactivation (2026-05-15)

This app was deactivated to cut hosting spend (not maintained, not income-generating). On Heroku: `maintenance:on`, `web` scaled to 0, `heroku-postgresql:essential-0` add-on destroyed. App config and custom domains (`dokodemofit.app`, `www.dokodemofit.app`) are preserved. ACM-managed SSL cert will lapse around 2026-06-29 because Let's Encrypt's HTTP-01 challenge needs a running web dyno — reactivation triggers a fresh cert within minutes.

**Reactivation procedure, DB dump, and config snapshot live in `~/code/chifury/heroku-backups/` (kept outside the repo because the config file contains secrets). See that directory's `README.md` for exact steps.**
