# Running WatchWeave Locally with Flox

This guide gets WatchWeave (frontend, backend, recommendation engine,
Postgres, Redis) running on your machine using Flox instead of Docker
Compose. No Docker required.

Tested on macOS (Apple Silicon) and Debian/Ubuntu Linux. Should work on
Windows via WSL2 too, per Flox's documented support.

## Prerequisites

- [Flox installed](https://flox.dev/docs/install-flox/install/)
- Git

That's it. You do **not** need Node, Python, Postgres, or Redis installed
on your machine beforehand — Flox provides all of these.

## Steps

### 1. Clone the repo

```bash
git clone https://github.com/lance-decandia/watchweave.git
cd watchweave
```

### 2. Activate the environment

```bash
flox activate -s
```

That's it — this one command does everything:
- Downloads Node 20, Python 3.11, Postgres 15, and Redis 7.2.7
- Creates a Python virtual environment (Flox's Python install doesn't
  bundle pip by default)
- Initializes Postgres and creates the `watchweave` database
- Installs npm dependencies for the backend and frontend
- Installs pip dependencies for the recommendation engine
- Starts all five services: Postgres, Redis, backend, recommendation
  engine, and frontend

The first run on a new machine takes a few minutes, since it's doing
real, one-time setup work (downloading runtimes, installing
dependencies). Every activation after that is fast, since everything is
cached and the setup steps are skipped once they're no longer needed.

### 3. Open the app

Visit `http://localhost:3001` in your browser.

Your local Postgres database starts **empty** — register a new test
account; logging in with credentials from the live watchweave.watch site
won't work, since that's a completely separate production database.

### 4. Check on things, if needed

```bash
flox services status
```
Shows the status of all five services.

```bash
flox services logs backend
flox services logs recommendation-engine
flox services logs frontend
```
Shows each service's output, useful if something doesn't start cleanly.

## What's different from `docker-compose.yml`

This Flox setup covers the same five core pieces as
`docker-compose.yml` (Postgres, Redis, backend, recommendation engine,
frontend) without needing Docker installed. It does **not** currently
cover Prometheus or Grafana — those still require Docker Compose
separately if you want local monitoring. Floxifying monitoring too is an
open idea for a future session.

## Troubleshooting

| Problem | Fix |
|---|---|
| Port 5000 already in use (macOS) | Disable AirPlay Receiver in System Settings → General → AirDrop & Handoff |
| Services don't show as running | Run `flox services status`, then `flox services logs <name>` to see why |
| Login fails with known-good credentials | You're on a fresh local DB — register a new account instead |
| First activation is slow | Expected — it's downloading runtimes and installing dependencies for the first time. Subsequent activations are fast. |

## Alternative: pulling the environment from FloxHub instead of git

The `.flox/` folder is committed in this repo, so the steps above are
the simplest path — one clone gets you both the code and the
environment together.

If you'd rather pull the environment directly from FloxHub instead of
relying on the one committed in the repo (for example, to always get
the latest pushed version, independent of what's committed to a
particular git branch), you can do that too:

```bash
git clone https://github.com/lance-decandia/watchweave.git
cd watchweave
rm -rf .flox
flox pull lance-decandia/watchweave
flox activate -s
```

This replaces the git-committed environment with one pulled fresh from
FloxHub, combined with the code already sitting in the cloned folder.
Confirmed working identically to the default path — tested by pulling
this exact environment from a separate FloxHub account with no prior
access to this repo.

You can also pull just the environment on its own, without any code, to
inspect or reuse it independently:
```bash
flox pull lance-decandia/watchweave --copy
```
The `--copy` flag gives you a fully independent copy you can edit and
push under your own account, detached from the original.

Redis defaults to its newest catalog version (8.x) unless explicitly
pinned — this manifest pins it to `7.2.7` to match production
(ElastiCache Redis 7) and `docker-compose.yml` (`redis:7-alpine`).

Postgres on Linux needs an explicit `unix_socket_directories` setting,
since the default (`/run/postgresql/`) doesn't exist on minimal
installs. This manifest points it at the Postgres data directory itself
instead, which works correctly on both macOS and Linux.
