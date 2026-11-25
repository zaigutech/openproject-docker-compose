# AGENTS.md

This repo is a fork of the upstream OpenProject Compose setup; keep the diff against upstream small.

- `docker-compose.yml` holds only configuration shared by development and production (shared infra, service names). Development defaults go in `docker-compose.override.yml`, production values in `docker-compose.prod.yml` (via `-f`), notes in `docs/`.
- Development and production both read secrets and machine settings from `.env`; never hardcode a dev secret to make a start without `.env` work.
- Leave other upstream files (`.env.example`, `README.md`) untouched, and avoid edits next to lines upstream bumps often, such as the hocuspocus image.
- Upstream's `.gitignore` ignores `docker-compose.override.yml`, so it needs `git add -f`.
