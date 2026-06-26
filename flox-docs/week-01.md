# Week 1 — Onboarding Journal

(Entries below are retroactive, written same-day. Going forward: real-time
entries during the day, full write-up during the last hour before signing
off.)

---

[2026-06-25, evening] [setup friction] [FLOX] 🔴
What happened: Installed python311 via [install]. Tried pip install -r
requirements.txt for the recommendation engine and got
"command not found: pip". Tried python3 -m pip — "No module named pip".
Expected vs. got: Expected pip to come bundled with Python, the way it
normally does with python.org installers, pyenv, or most Docker images.
Resolution: python3 -m ensurepip failed with
"error: externally-managed-environment" — Nix's /nix/store is immutable
by design, won't let pip-like tools write into it. Created a venv via
[hook] (python3 -m venv $FLOX_ENV_CACHE/python-venv) instead. pip works
fine inside the venv since it's a normal writable directory.
Time cost: ~20 min.
Notes: Real first-contact gotcha for anyone used to pyenv/conda/Docker
workflows where pip is just always there.

---

[2026-06-25, evening] [confusing error] [FLOX] 🟡
What happened: Tried to connect to local Postgres with psql ... -d
watchweave after a clean initdb. Got
"FATAL: database watchweave does not exist".
Expected vs. got: Expected initdb to also create my app's named
database, the way Docker's official postgres image does automatically
via the POSTGRES_DB env var.
Resolution: initdb only creates the cluster (postgres, template0,
template1) — not app-specific databases. Had to add an explicit
createdb step to the [hook], temporarily starting Postgres, checking
if the db exists, creating it if not, then stopping it again (since
[services] starts it properly afterward).
Time cost: ~15 min.
Notes: Worth flagging as a docs gap — nothing warns you that initdb !=
"fully set up like Docker's postgres image."

---

[2026-06-25, evening] [expected X got Y] [FLOX] 🟢
What happened: flox show redis listed 8.8.0 as "Latest," but running
unversioned redis.pkg-path = "redis" resolved to 8.6.3 instead — and
my project needs Redis 7 to match docker-compose.yml and production
(ElastiCache Redis 7).
Resolution: Explicitly pinned redis.version = "7.2.7" in [install].
Confirmed via redis-server --version after re-activating.
Time cost: ~10 min.
Notes: Didn't fully resolve why unpinned didn't match the catalog's own
"Latest" label — possibly a stability-tier thing, not confirmed. Lesson:
always pin explicitly, don't trust unpinned defaults to match what you'd
expect.

---

[2026-06-25, evening] [setup friction] [SYS] 🟢
What happened: Recommendation engine (Flask, port 5000) failed with
"Address already in use."
Resolution: macOS's AirPlay Receiver uses port 5000 by default. Disabled
it in System Settings.
Time cost: ~10 min.
Notes: Not Flox's fault at all — known macOS quirk, independent of this
project. Worth documenting in the walkthrough anyway since anyone on Mac
will likely hit it.

---

[2026-06-25, evening] [missing feature] [FLOX] 🔴
What happened: Sourced the venv's activate script inside [hook], and
$VIRTUAL_ENV was set correctly, pip/Flask showed installed correctly —
but python app.py still failed with ModuleNotFoundError: No module
named 'flask'.
Expected vs. got: Expected the venv to fully take over python/pip
resolution once sourced.
Resolution: which python3 showed it was still resolving to Flox's own
Nix-provided Python, not the venv's, despite the venv being "active."
Checked $PATH order — venv's bin/ was present but positioned after
Flox's own bin/ directories, so it never won. Root cause: [hook].on-
activate runs before Flox's own final $PATH assembly; anything done there
can get overridden. Fix: moved the venv activation (not creation) into
[profile], which runs later in the sequence — confirmed in Flox's own
docs as the documented-correct place for "sourcing the activate script
for a Python virtual environment." Re-tested: which python3 now
correctly resolves to the venv, plain python app.py works with no
workaround needed.
Time cost: ~40 min total (including the initial discovery + the later
proper fix).
Notes: This is the most interesting finding of the night. Genuinely
useful for the team — the hook-vs-profile distinction for anything
touching $PATH is non-obvious and easy to get wrong on a first attempt.

---

[2026-06-25, evening] [setup friction] [FLOX] 🟢
What happened: Forgot Docker Desktop isn't installed on this (new)
machine, so couldn't test whether Grafana/Prometheus still work against
Flox-run backend/recommendation-engine services.
Resolution: Deferred — confirmed this is genuinely out of scope for
Project 1's "done" criteria (monitoring was never required), and would
need a separate Docker Desktop install to test properly. Logged as a
follow-up rather than blocking tonight's work.
Time cost: ~5 min (just the realization, no further debugging).
Notes: Good finding in itself — got the entire core app (Postgres,
Redis, backend, frontend, recommendation engine) running via Flox with
zero Docker installed at all. Stronger proof of Flox's value than
originally expected, since this wasn't "Flox vs already-working Docker
setup" — it was "Flox from a completely blank machine."

---

[2026-06-26, cross-platform testing] [confusing error] [FLOX] 🔴
What happened: Tested the manifest on a fresh Debian VM (separate
machine, separate Flox account). flox activate -s ran the hook
successfully, but flox services status showed postgres as
"Completed (1)" instead of "Running" — connection refused on psql.
Resolution: flox services logs postgres showed the real error:
could not create lock file "/run/postgresql/.s.PGSQL.5432.lock":
No such file or directory. This directory exists by default on
typical Debian/Ubuntu installs (created by the postgres apt package's
own setup), but never existed on this minimal VM, since Postgres was
installed entirely through Flox/Nix instead of apt. Fixed by explicitly
setting -c unix_socket_directories="$PGDATA" in the postgres service
command, pointing it at a directory Flox itself creates and controls,
rather than depending on a system path that may or may not exist.
Time cost: ~25 min.
Notes: This never showed up on macOS, which doesn't use the same
/run/postgresql/ convention. A great example of a hidden, untested
assumption — the manifest "worked" on Mac by accident, not because the
underlying logic was actually platform-independent.

---

[2026-06-26, cross-platform testing] [missing feature] [FLOX] 🟡
What happened: After fixing the lock-file issue above and reactivating,
postgres started running correctly, but psql -d watchweave still said
the database didn't exist.
Resolution: Found the actual bug: the createdb check was nested inside
the same if [ ! -d "$PGDATA" ] block as initdb. Since $PGDATA had
already been created by the first, failed attempt (the one that hit
the lock-file bug above), the whole block — including createdb — got
skipped on every later activation, even after the lock-file bug was
fixed. Un-nested createdb into its own independent check so it verifies
the database exists on every activation, regardless of whether $PGDATA
already existed from a prior attempt.
Time cost: ~15 min.
Notes: This bug existed on Mac too the whole time — it just never
surfaced there, because the Mac's first successful activation happened
to include both initdb and createdb together, in one clean pass, before
$PGDATA existed yet. The Linux lock-file bug accidentally exposed a
second, unrelated bug that had been latent since the start.

---

[2026-06-26, redesign] [setup friction] [FLOX] 🟢
What happened: After getting everything working on a second platform,
realized how repetitive and slow it was to manually cd into three
separate folders (backend, frontend, recommendation-engine) and run
npm install / pip install / npm start / python app.py by hand, across
multiple terminal tabs, on every single machine.
Resolution: Restructured the manifest so backend, recommendation-engine,
and frontend are real Flox services (alongside postgres and redis), and
moved their dependency installs into [hook] with conditional checks
(only installing if node_modules/the venv's packages don't already
exist). Now a single flox activate -s brings up the entire five-piece
stack automatically — no manual per-service steps at all.
Time cost: ~30 min to design, write, and test on both Mac and Linux.
Notes: This is probably the single most valuable change of the whole
night for actual usability. The original manual-steps approach
technically worked and was "correct" per the docs/wizard defaults, but
real friction from doing it three+ times across multiple machines made
it clear the convenience tradeoff was worth the extra hook complexity.
Worth raising with Rok whether this pattern (services for app
processes, not just databases) should be the default recommendation in
Flox's own onboarding docs.

---

[2026-06-26, cross-platform testing] [setup friction] [SYS] 🟢
What happened: Fresh Debian VM had neither curl nor git installed by
default.
Resolution: sudo apt install -y curl git. Trivial, but a real first
hurdle on a genuinely minimal Linux install.
Time cost: ~5 min.
Notes: Worth remembering for the walkthrough's "prerequisites" section
— "Flox and Git" assumes git already exists, which isn't universally
true on minimal installs.

---

End of week 1 reflection: tonight ended up being a much deeper
validation than originally planned — full cross-platform testing
(macOS, Debian, Ubuntu), two real bugs found and fixed at the root
cause (not just patched), and a full redesign of how the app's own
services are managed. The Rok quote about breaking things to learn the
why, not just the how, kept proving itself true in practice — almost
every real insight tonight came from something failing first, not from
reading docs in advance.