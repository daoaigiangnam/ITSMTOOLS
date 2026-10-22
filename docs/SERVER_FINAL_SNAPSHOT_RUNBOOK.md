# ITSMTOOLS — Server Final Snapshot Runbook

Date: 2026-10-22
Target branch: release/server-final-2026-10-22
Server application path: /www/wwwroot/kpi.review360.id.vn
Framework: Laravel 12 / PHP 8.2

## Purpose
Capture and review the exact production source on the aaPanel server before publishing it to ITSMTOOLS. Do not assume the server filesystem matches the existing kpi_dashboard GitHub branch.

## Production safety rules
- Do not run git reset --hard, git clean -fd, or checkout over the live application.
- Never publish .env, real credentials/API tokens, production SQL dumps, storage logs, sessions, cache, or runtime uploads.
- Back up code and database separately first.
- Do not run migrations against production as part of source packaging.
- Keep the current Nginx document root at /www/wwwroot/kpi.review360.id.vn/public.
- Keep production PHP at 8.2 unless dependencies have been deliberately qualified for a different version.

## Step 1 — Read-only server diagnostics
Run from the aaPanel terminal:

```bash
cd /www/wwwroot/kpi.review360.id.vn
printf '\n===== PHP =====\n'; php -v | head -n 2
printf '\n===== LARAVEL =====\n'; php artisan --version
printf '\n===== GIT STATUS =====\n'; git status --short --branch
printf '\n===== GIT REMOTE =====\n'; git remote -v
printf '\n===== LAST COMMIT =====\n'; git log -1 --format='%H%n%ci%n%s' 2>/dev/null || true
printf '\n===== DEPENDENCIES =====\n'; grep -nE '"php"|"laravel/framework"|"phpoffice/phpspreadsheet"' composer.json
printf '\n===== ROUTE VALIDATION =====\n'; php artisan route:list --except-vendor
```

Save the output before proceeding. If there are modified or untracked files, preserve them and compare them; do not reset or clean them away.

## Step 2 — Backup
Choose a protected backup destination outside the web root, with sufficient free space. Confirm backup permissions and retention before doing this.

Example code archive command (creates a source-only archive and excludes secrets/runtime files):

```bash
cd /www/wwwroot
tar --exclude='kpi.review360.id.vn/.git' \
    --exclude='kpi.review360.id.vn/.env' \
    --exclude='kpi.review360.id.vn/.env.*' \
    --exclude='kpi.review360.id.vn/vendor' \
    --exclude='kpi.review360.id.vn/node_modules' \
    --exclude='kpi.review360.id.vn/storage/logs/*' \
    --exclude='kpi.review360.id.vn/storage/framework/cache/data/*' \
    --exclude='kpi.review360.id.vn/storage/framework/sessions/*' \
    --exclude='kpi.review360.id.vn/storage/framework/views/*' \
    -czf /path-in-protected-backup/kpi-server-source-2026-10-22.tar.gz \
    kpi.review360.id.vn
```

Replace /path-in-protected-backup with an actual protected backup path outside the web root. This archive command is a starting point: review the archive listing to ensure storage contains no customer uploads or secrets before using it.

Database backup must be created separately using the actual DB connection credentials and host from the server's private .env; do not paste credentials into chat or commit them. Protect and verify the resulting SQL dump.

## Step 3 — Review Git-eligible source
Before publishing, check at minimum:
- .gitignore excludes .env files, /vendor, /node_modules, logs and local cache.
- No API keys, SMTP passwords, customer information, production database dumps, access tokens, or real account passwords appear in tracked files.
- Review README for default credentials; do not enable or publish a known default production account.
- Verify migrations are additive and compatible; do not include a production data dump.
- Run route:list and tests in a staging copy, not against a live database.
- For Laravel runtime directories, preserve only required .gitkeep placeholders.

Useful secret scan (review matches manually; it may report harmless strings too):
```bash
cd /www/wwwroot/kpi.review360.id.vn
git status --short
git ls-files | grep -E '(^|/)\.env($|\.)|\.sql$|\.dump$|\.bak$|storage/logs|storage/framework'
```

## Step 4 — Publish to ITSMTOOLS (do only after reviewed snapshot is ready)
Preferred workflow:
1. Create a clean staging checkout/copy outside the live web root.
2. Copy the reviewed source from the server snapshot into staging, respecting .gitignore and Laravel directory conventions.
3. Initialize a fresh Git history if this is intended to be an independent product repository.
4. Add remote https://github.com/daoaigiangnam/ITSMTOOLS.git.
5. Checkout/create release/server-final-2026-10-22 and commit the reviewed source.
6. Push the branch; never push .env or the database dump.
7. Open a pull request to main only after smoke tests, permission tests, migration review, and secret scan pass.

## Server deployment facts from aaPanel screenshots
- Project root: /www/wwwroot/kpi.review360.id.vn
- Nginx root: /www/wwwroot/kpi.review360.id.vn/public
- Domains: kpi.review360.id.vn and service.review360.id.vn
- PHP configured in aaPanel: PHP-82
- Database shown in aaPanel: kpi_dashboard / user kpi_dashboard
- MySQL server shown in aaPanel: 8.0.45

Confirm the live .env connection and actual DB server version locally. Do not infer the exact runtime configuration from screenshots alone.
