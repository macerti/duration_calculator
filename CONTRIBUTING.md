# Contributing — start here

New to this project? Read in this order:

1. **`ORIENTATIONS.md`** — the standing rules for *how* we build and ship
   here (logging files, deployment shape, security discipline, UI tokens).
   Read this first, before touching any code.
2. **`README.md`** — what this folder is and how it's deployed.
3. **`DEPLOY.md`** — the exact DirectAdmin deploy steps (also automated by
   CI, see below — but read this to understand what CI is doing on your behalf).
4. **`SECURITY.md`** — current security posture, what's fixed, what's open.
5. **`ROADMAP.md`** / **`BUGLOG.md`** / **`CHANGELOG.md`** — project history
   and what's next. Check these before starting work so you're not
   re-litigating a settled decision or re-fixing a logged bug.
6. **`TEST_CHECKLIST.md`** — manual test scenarios; add a Test History entry
   whenever you do a pass.

## Local setup

- This repo is the **deployable artifact** — frontend static build +
  PHP/MySQL backend — not a buildable frontend source project. There is no
  `npm install` here (no `package.json`).
- Copy `config.example.php` → `config.php` and fill in your own local DB
  credentials. **`config.php` is gitignored — never commit it** (it holds a
  real DB password once filled in for production).
- Backend needs PHP 8.0+ and a MySQL/MariaDB database (`db/schema.sql`).
- Run the engine test suite locally: `php tests/smoke_test.php` — this only
  exercises the calculation engine (`engine/`, `data/`), no DB/config needed.

## Making a change

- Every code change ships together with, in the same commit/PR: a
  `CHANGELOG.md` entry (bump `x.y.z` per the convention in
  `ORIENTATIONS.md`), a `BUGLOG.md` entry if it's a bug fix, and updated
  `SECURITY.md`/`TEST_CHECKLIST.md` if the change touches input handling,
  a new endpoint, or new stored data.
- If you change frontend behavior, remember the actual React/Expo **source**
  lives outside this deployable folder (see "Rebuilding after changes" in
  `README.md`) — this repo only holds the already-built static output plus
  the PHP backend. Rebuild there, then copy the static output over.
- Never commit `config.php`, `.env`, or any file with real credentials.

## Deployment

Pushing to `main` (or running the workflow manually from the **Actions** tab)
runs `.github/workflows/deploy.yml`: it runs the PHP engine tests, then
FTPS-deploys everything except `config.php` to
`tools.macerti.com/duration_calculator/`. The real `config.php` lives only on
the server and is never touched by the deploy.

Live app: **https://tools.macerti.com/duration_calculator/**
