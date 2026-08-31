# Deploying to DirectAdmin — tools.macerti.com/duration_calculator/

This is one self-contained folder. Extract, upload as-is, run a few one-time
setup steps. No build step happens on the server — the frontend is already
built and pointed at this exact URL.

## 1. Confirm PHP version

DirectAdmin → **Select PHP Version** (or similar). Needs **PHP 8.0+**, ideally
8.1/8.2.

## 2. Create the subdomain (if not already done)

DirectAdmin → Subdomains → create `tools`. This gives it its own document
root (e.g. `public_html/tools/` or similar — check what DirectAdmin assigns).

## 3. Upload

Extract this zip, upload the entire `duration_calculator/` folder into that
subdomain's document root, so the path on disk is
`.../tools_docroot/duration_calculator/`. This is where future tools would
live as siblings, e.g. `.../tools_docroot/another_tool/`.

## 4. Run the database schema

phpMyAdmin (DA PMA SignOn) → `macerti_audit_calc` → SQL tab → paste the
contents of `db/schema.sql` → run. Creates 3 tables: `parameter_sets`,
`calculation_cases`, `parameter_change_log`.

## 5. Seed the parameter set

With SSH:
```bash
cd .../duration_calculator && php seed.php
```

Without SSH: temporarily create `api/seed-once.php` containing
```php
<?php require __DIR__ . '/../seed.php';
```
visit it once in your browser, confirm the success message, then **delete
that file**.

## 6. Verify the backend

Visit `https://tools.macerti.com/duration_calculator/api/health`. Expect:
```json
{"status":"ok","parameterSetId":"default-v1","version":1,"dbConnected":true,"dbBackedParameters":true}
```
- `dbConnected: false` → check `config.php` is present and its credentials
  are correct.
- `dbConnected: true` but `dbBackedParameters: false` → you skipped step 5.

## 7. Lock down config.php

`chmod 600 config.php` (via SSH or File Manager's permissions dialog) — it
holds your DB password in plain text.

## 8. Confirm sensitive files are actually blocked

Visit these directly in a browser — each should show a **403 Forbidden**,
not the raw file:
- `https://tools.macerti.com/duration_calculator/db/schema.sql`
- `https://tools.macerti.com/duration_calculator/data/raw/nace_risque_table.csv`

If you get the raw file content instead of a 403, `.htaccess` isn't being
honored — ask your host to confirm `AllowOverride All` is set for that
directory (this is the most common shared-hosting gotcha for `.htaccess`).

## 9. Open the app and test a real calculation

`https://tools.macerti.com/duration_calculator/` → try the Case Calculator
end to end. This confirms the frontend can actually reach the backend, not
just that both load independently.

## 10. Wire up the WordPress button

Point it at `https://tools.macerti.com/duration_calculator/`.

## 11. Tighten CORS (optional, recommended once confirmed working)

Edit `config.php`:
```php
'allowedOrigins' => ['https://www.macerti.com', 'https://macerti.com'],
```
Not strictly required since the frontend and API share an origin
(`tools.macerti.com`), but tightens things if you ever call the API directly
from `www.macerti.com`.

## Rebuilding after future changes

Only needed if the frontend code changes. From the `frontend/` source project:

```bash
cd frontend
npm install
EXPO_PUBLIC_API_URL=https://tools.macerti.com/duration_calculator/api npx expo export --platform web --clear
```

`app.json` already has `"experiments": { "baseUrl": "/duration_calculator" }`
set — this is what makes `index.html`'s asset references (`<script src>`,
favicon) correctly point at `/duration_calculator/...` instead of the domain
root. If you ever rename the deployed folder, update both this `baseUrl` and
`EXPO_PUBLIC_API_URL` to match, and rebuild.

⚠️ Always include `--clear` — Metro's bundler cache can otherwise silently
reuse a stale build with an old URL baked in.

Copy `frontend/dist/*` over everything in this folder **except**: `api/`,
`engine/`, `data/`, `db/`, `config.php`, `seed.php`, `tests/`, `.htaccess`.

## Troubleshooting

- **Blank white page**: almost always a stale/wrong asset base path. Open
  browser dev tools → Network tab → look for 404s on `.js`/`.css`/`.ico`
  files, check what URL they actually requested vs. what it should be.
- **404 on every `/api/...` request**: `.htaccess` rewrite isn't active —
  same `AllowOverride`/`mod_rewrite` check as step 8.
- **500 everywhere under `/api/`**: check DirectAdmin's PHP error log —
  usually `config.php` missing or malformed, or PHP version below 8.0.
- **CORS errors in the console**: frontend's origin isn't in `allowedOrigins`
  — see step 11.


## Mandatory source/deployment separation

This repository is the **DEPLOYMENT ARTIFACT repository**, not the application source repository. The application source of truth is **macerti/duration_calculator_backend**.

- Do not develop or permanently fix application logic directly in this repository.
- Buildable source changes originate in macerti/duration_calculator_backend.
- The generated Expo web artifact and deploy-ready PHP tree must be published here after source changes are tested.
- The canonical build path is the manual **Build deploy artifact from source** workflow in .github/workflows/build-from-source.yml. It checks out source main, runs the Expo web export with the production API URL, copies the generated web artifact into this repository, and commits the result.
- PHP files at repository root are the deployable projection of duration-calculator-php/ from the source repository.
- A deployment commit must never be described as a source fix unless the corresponding source commit exists in macerti/duration_calculator_backend.
- Every hand-off must record the source commit and this repository's artifact commit.
- If source and artifact differ, source is authoritative; regenerate rather than hand-editing the artifact.
