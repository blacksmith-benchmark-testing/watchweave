# Recurring Themes (running summary, updated weekly)

This file is the seed for the week 9 prioritized memo. Add 2-3 lines at
the end of each week — don't wait until the end to synthesize everything
cold.

---

## Week 1

- Nix's Python is intentionally minimal (no pip) — anyone porting a
  Python project to Flox without prior Nix experience will hit this.
  Worth a clearer callout in Flox's own Python onboarding docs.
- Unpinned packages don't necessarily resolve to the catalog's literal
  "Latest" version — pin explicitly, always, especially for anything
  where version compatibility matters (Redis, in this case).
- initdb creates the Postgres cluster, not app-specific databases —
  unlike Docker's official postgres image, which auto-creates via
  POSTGRES_DB. A real gap vs. expectations coming from Docker.
- [hook].on-activate runs before Flox's final $PATH assembly — anything
  there that touches $PATH (like sourcing a venv) can get silently
  overridden. [profile] is the documented-correct place instead, but
  this distinction is easy to miss on a first attempt.
- Cross-platform testing (Mac, Debian VM, Ubuntu VM) surfaced two real,
  previously-invisible bugs: Postgres on Linux needs an explicit
  unix_socket_directories setting (no /run/postgresql/ on minimal
  installs), and a createdb check was wrongly coupled to the initdb
  directory check, so it silently never re-ran after any failed first
  attempt. Both bugs existed on every platform from the start — Linux
  just happened to be where the right (wrong) sequence of events
  exposed them.
- Biggest design lesson: manually running npm/pip install and starting
  each service by hand, repeated across multiple terminals and
  machines, was real, deserved friction — not a misunderstanding of how
  Flox works. Solved by making backend/recommendation-engine/frontend
  real Flox services too, with conditional installs in the hook, so
  one flox activate -s does everything. Worth considering whether
  this should be the default pattern Flox recommends for multi-service
  local dev, not just databases/caches.
- Biggest win: got the full WatchWeave stack (5 services) running
  locally via Flox alone, validated end-to-end on three separate
  machines/platforms, with zero Docker installed on two of them.