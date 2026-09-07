# Spec: Registration

## Overview

This step implements the actual account-creation logic behind the existing `/register` page. Today `GET /register` only renders `register.html` — the form posts to `/register` but there is no route to receive it. This feature adds `POST /register`, which validates the submitted name/email/password, hashes the password, inserts a new row into `users`, and gets the user into a logged-in session so they land somewhere useful after signing up. This is the first authentication step built on top of the database layer from Step 1, and every later step (login, logout, profile, expenses) depends on users being able to register and on a session mechanism existing.

## Depends on

- Step 1 (Database setup) — requires `get_db()`, `init_db()`, and the `users` table with `email UNIQUE` and `password_hash` columns.

## Routes

- `POST /register` — accepts `name`, `email`, `password` form fields, validates and creates the user, starts a session — public
- `GET /logout` **is not** in scope for this step (remains the Step 3 stub) — but registration must set `session["user_id"]` in a way the future logout step can clear

## Database changes

No database changes. `database/db.py` already defines `users(id, name, email, password_hash, created_at)` with `email UNIQUE NOT NULL`. Registration will reuse this table via new helper functions — no schema changes needed.

New DB helper functions to add to `database/db.py` (logic only, not schema):
- `get_user_by_email(email)` — `SELECT * FROM users WHERE email = ?`, used to check for duplicate emails before insert
- `create_user(name, email, password_hash)` — parameterized `INSERT INTO users (...) VALUES (?, ?, ?)`, returns the new user id

## Templates

- Create: none
- Modify: `templates/register.html` — repopulate submitted `name`/`email` values on validation failure so the user doesn't retype everything (e.g. `value="{{ name or '' }}"`)

## Files to change

- `app.py` — add `POST` to the `/register` route, add `session` handling (`app.secret_key` must be set), import `get_user_by_email` / `create_user` from `database/db.py`
- `database/db.py` — add `get_user_by_email()` and `create_user()` functions
- `templates/register.html` — repopulate submitted values on error

## Files to create

- None

## New dependencies

No new dependencies. `werkzeug.security` (`generate_password_hash`) is already used in `database/db.py`; Flask's built-in `session` covers login state.

## Rules for implementation

- No SQLAlchemy or ORMs
- Parameterized queries only (`?` placeholders) — never f-strings in SQL
- Passwords hashed with `werkzeug.security.generate_password_hash` before storing — never store plaintext
- Use CSS variables — never hardcode hex values (reuse existing `auth-*` classes in `style.css`, no new inline styles)
- All templates extend `base.html`
- Validate on the server even though the form has `required`/`type=email` attributes client-side (name non-empty, valid-looking email, password length ≥ 8)
- On duplicate email, re-render `register.html` with an `error` message and HTTP 200 (not a redirect) — must not crash
- `app.secret_key` must be set (e.g. from an env var with a dev fallback) for `session` to work
- Route function stays thin: parse form → validate → call `database/db.py` helpers → set session → redirect; no raw SQL inside `app.py`

## Definition of done

- [ ] Submitting the register form with a new name/email/password creates a row in `users` with a hashed (not plaintext) password
- [ ] After successful registration, the user is redirected and `session` contains their user id
- [ ] Submitting with an email that already exists in `users` re-renders `register.html` with an error message and does not create a duplicate row
- [ ] Submitting with a password under 8 characters is rejected server-side even if client-side `required`/`minlength` is bypassed
- [ ] Submitting with an empty name or malformed email is rejected server-side
- [ ] No SQL in `app.py` — all queries live in `database/db.py` and use `?` placeholders
- [ ] `python app.py` starts on port 5001 without errors and `/register` still renders on `GET`
