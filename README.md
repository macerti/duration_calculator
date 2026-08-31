# Audit Duration Calculator — Deployable Single Folder

This is one self-contained folder: extract it, upload it as-is to
`tools.macerti.com/duration_calculator/`, and it works — frontend (static
files) and backend (PHP API) both live inside `duration_calculator/`.

```
duration_calculator/
├── index.html, _expo/, assets/, favicon.ico, metadata.json   ← frontend (static)
├── api/index.php, api/.htaccess                              ← the one PHP entrypoint
├── engine/, data/, db/                                        ← PHP calculation engine
├── config.php                                                 ← your real DB credentials (already filled in)
├── seed.php                                                    ← run once to activate the parameter set
├── tests/smoke_test.php                                        ← 24 checks against the GS0106 spec's worked examples
└── .htaccess                                                    ← blocks direct access to .php/.sql/.csv outside api/
```

External URLs once deployed:
- `https://tools.macerti.com/duration_calculator/` — the app itself
- `https://tools.macerti.com/duration_calculator/api/health` — backend health check
- `https://tools.macerti.com/duration_calculator/api/calculate` — the actual calculation endpoint (POST)

## You do not build anything on the server

PHP needs no build step — it just runs. The frontend **was already built**
for this exact URL (`EXPO_PUBLIC_API_URL=https://tools.macerti.com/duration_calculator/api`)
and the static output is already sitting in this folder. You're extracting
and uploading, not compiling anything server-side.

If you ever change a screen and need to rebuild the frontend yourself later,
see "Rebuilding after changes" below — but for this delivery, it's done.

## Deploy steps

1. **Check PHP version** in DirectAdmin (Select PHP Version / similar) — needs
   8.0+, ideally 8.1/8.2.
2. **Create the subdomain** `tools.macerti.com` if you haven't already
   (DirectAdmin → Subdomains), pointing at whatever folder DirectAdmin gives
   it by default (e.g. `public_html/tools` or similar — this becomes the
   subdomain's document root, and will hold `duration_calculator/` as a
   subfolder, plus any future tools alongside it).
3. **Upload**: extract this zip, upload the entire `duration_calculator/`
   folder into that subdomain's document root, so the path on disk is
   `.../tools_docroot/duration_calculator/`.
4. **Run the schema**: phpMyAdmin → `macerti_audit_calc` → SQL tab → paste
   `db/schema.sql` → run. Creates 3 tables.
5. **Seed the parameter set**:
   - With SSH: `cd .../duration_calculator && php seed.php`
   - Without SSH: temporarily create `api/seed-once.php` containing
     `<?php require __DIR__ . '/../seed.php';`, visit it once in your
     browser, confirm the success message, then **delete that file**.
6. **Verify**: visit `https://tools.macerti.com/duration_calculator/api/health`
   — expect `"dbConnected":true,"dbBackedParameters":true`. If `dbConnected`
   is `false`, check that `config.php` is present and readable. If it's `true`
   but `dbBackedParameters` is `false`, you skipped step 5.
7. **Lock down `config.php`**: set file permissions to `600` or `640` via
   File Manager or SSH (`chmod 600 config.php`) — it has your DB password.
8. **Open the app**: `https://tools.macerti.com/duration_calculator/` — try
   the NAE Calculator or Case Calculator screens to confirm the frontend can
   actually reach the backend (not just that both load independently).
9. **Wire up the WordPress button**: point it at
   `https://tools.macerti.com/duration_calculator/` — done.
10. **Tighten CORS** (optional but recommended once confirmed working): edit
    `config.php`, change `'allowedOrigins' => ['*']` to
    `'allowedOrigins' => ['https://www.macerti.com', 'https://macerti.com']`.
    This isn't strictly needed since the frontend and API are same-origin
    (`tools.macerti.com`), but tightens things in case you ever call the API
    from `www.macerti.com` directly (e.g. a WordPress widget).

## Rebuilding after changes

Only needed if you (or I, later) change frontend code. From the `frontend`
source project (separate from this deployable folder):

```bash
cd frontend
npm install
EXPO_PUBLIC_API_URL=https://tools.macerti.com/duration_calculator/api npx expo export --platform web --clear
```

⚠️ Always include `--clear` — without it, Metro's bundler cache can silently
reuse a stale build with the wrong API URL baked in (this bit us once during
testing; see `BUGLOG.md` in the source project).

Then copy `frontend/dist/*` over the static files in this folder (everything
except `api/`, `engine/`, `data/`, `db/`, `config.php`, `seed.php`, `tests/`,
`.htaccess`) and re-upload just those changed static files.

## Troubleshooting

- **404 on every `/api/...` request**: `.htaccess` isn't being honored —
  confirm `AllowOverride All` for that directory (ask your host if unsure)
  and `mod_rewrite` is enabled (usually on by default).
- **Blank page / 500 everywhere**: check DirectAdmin's PHP error log —
  almost always `config.php` missing, or PHP version below 8.0.
- **App loads but every action fails**: open browser dev tools → Network tab
  → check what URL the failing request actually went to. Should be
  `.../duration_calculator/api/...`. If it's pointed somewhere else, the
  frontend was built with the wrong `EXPO_PUBLIC_API_URL`.

## Working on this project

See `ORIENTATIONS.md` for standing technical/process conventions (logging
convention, deployment shape, testing standard) that apply regardless of
which feature is being worked on. See `CHANGELOG.md`, `ROADMAP.md`,
`BUGLOG.md` for this project's actual history.


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
