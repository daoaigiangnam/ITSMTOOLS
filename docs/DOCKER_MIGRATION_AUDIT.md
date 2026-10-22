# ITSMTOOLS — Production Discovery and Docker Migration Audit

Status: Discovery plan only. No production changes authorized by this document.
Target app: /www/wwwroot/kpi.review360.id.vn
Target Git branch: release/server-final-2026-10-22

## Objective
Inventory the actual running production system before designing Docker Compose. Do not infer live behavior from source code or screenshots alone.

## Audit inventory
1. Application source: Laravel/PHP versions, extensions, Composer lock, route list, migrations, writable paths, upload/export paths, Git dirty/untracked files.
2. Database: actual DB host/version, schemas, charset/collation, grants, size, migration status, backup and restore test.
3. Schedules: aaPanel Cron entries, root/www crontabs, /etc/crontab, /etc/cron.d, hourly/daily jobs, Laravel schedule:list, actual scheduler trigger, task logs and overlap behavior.
4. Background workers: Supervisor, systemd units/timers, Laravel queue workers, scheduled shell/Python/Node scripts, startup scripts and process managers.
5. Network/services: listening TCP/UDP ports, reverse proxies, Nginx vhosts, PHP-FPM, MySQL/Redis, Docker containers/compose projects, external APIs/webhooks/SMTP, DNS and TLS.
6. Data and persistence: storage/app uploads, generated exports, logs, sessions/cache, private files, symlinks, mounted disks, backups, file ownership and permissions.
7. Operations/security: disk/RAM/CPU, service logs, firewall rules, secrets locations, certificates, monitoring and restore procedures.

## Read-only commands
Run commands in small groups; save outputs in a protected audit folder. Redact passwords, tokens, private keys, cookie/session values and customer data before sharing.

### Host and ports
```bash
cat /etc/os-release
uname -a
uptime
df -h
free -h
ss -lntup
systemctl --type=service --state=running --no-pager
systemctl list-timers --all --no-pager
```

### Runtime and containers
```bash
php -v
php -m
cd /www/wwwroot/kpi.review360.id.vn && php artisan --version
docker ps -a
docker compose ls
ps -eo user,pid,ppid,lstart,args --sort=pid | grep -E '[p]hp|[n]ode|[p]ython|[s]upervisord|[q]ueue:work|[s]chedule:work'
```

### Cron and Laravel scheduler
```bash
crontab -l 2>&1
crontab -u root -l 2>&1
crontab -u www -l 2>&1
ls -la /etc/cron.d /etc/cron.hourly /etc/cron.daily 2>/dev/null
cat /etc/crontab
journalctl -u cron --since '7 days ago' --no-pager 2>/dev/null
journalctl -u crond --since '7 days ago' --no-pager 2>/dev/null
cd /www/wwwroot/kpi.review360.id.vn && php artisan schedule:list
supervisorctl status 2>&1
systemctl list-unit-files --type=service --no-pager | grep -Ei 'php|nginx|mysql|maria|redis|supervisor|queue|kpi|review|node|python'
```

aaPanel Cron jobs must also be inventoried from the aaPanel UI; not every panel-managed task is guaranteed to be visible in the currently selected user's crontab. Inspect relevant systemd unit files and Supervisor configs if discovered. Do not dump whole environment files or secrets.

### App and Git status
```bash
cd /www/wwwroot/kpi.review360.id.vn
git status --short --branch
git remote -v
git log -1 --format='%H%n%ci%n%s' 2>/dev/null || true
php artisan route:list --except-vendor
php artisan schedule:list
find storage/app -maxdepth 3 -type d -print 2>/dev/null
find storage -maxdepth 3 -type l -ls 2>/dev/null
```

Do not run git reset/clean, production migrations, service restarts, or destructive tests during discovery.

## Deliverables before Docker design
- Inventory table: component, current owner/manager, command/config path, dependencies, port, persistence, schedule, logs, criticality.
- Cron/scheduler matrix: exact command, user, frequency, timezone, overlap policy, timeout, logs, failure alert, proposed Docker service.
- Data map: database(s), uploads, generated files, sessions/cache, secrets and backup locations.
- Dependency map: public domains -> Nginx/TLS -> PHP app -> DB/cache/queue -> external APIs and mail.
- Backup/restore evidence and production smoke-test checklist.
- Difference report: live server vs GitHub source (commit, modified/untracked files, ignored production-only files).

## Proposed Docker design (finalize only after inventory)
Use Docker Compose with separate responsibilities where needed:
- web/proxy: Nginx; either retain aaPanel Nginx as host reverse proxy or migrate TLS/proxy deliberately, never bind conflicting host ports accidentally.
- app: PHP-FPM with pinned PHP 8.2 image initially, required extensions, Composer dependencies; mount/build public assets as appropriate.
- scheduler: one dedicated Laravel scheduler container running `php artisan schedule:work`, or a single cron service if explicitly chosen. Never run duplicate schedulers across replicas.
- worker: separate queue worker only if queues are actually configured/used; define retry, timeout and restart policy.
- database: keep external aaPanel MySQL initially if lower risk, or migrate to a dedicated MySQL 8 container only after backup/restore, compatibility and storage design are validated.
- redis: only if used and configured; make persistence decision explicitly.
- volumes: persistent DB data, user uploads/private files, and any durable generated outputs; avoid persisting caches unnecessarily.
- secrets: runtime environment file outside Git, restricted permissions; no secrets in image layers.
- health checks, restart policies, log rotation, resource limits, backup jobs and restore documentation.

## Migration phases and go/no-go gates
1. Discovery only; no changes to production.
2. Code + database + uploads backup; validate backup integrity and restore to isolated staging.
3. Build images and compose files on a separate staging path/host, using sanitized configuration and test data.
4. Compare routes, permissions, exports/imports, uploads, email, integrations, scheduler effects and queue processing against current production.
5. Run schedules in dry-run/disabled mode during initial staging to prevent duplicate external side effects.
6. Cutover only after owner approval: maintenance window, final database delta/backup, controlled DNS/reverse-proxy switch, smoke tests and monitoring.
7. Rollback plan: retain old code, DB snapshot, uploads and known-good Nginx config; define exact trigger and restore sequence.

Do not move production to Docker until all scheduled jobs and persistent data paths have an explicit owner and test evidence.
