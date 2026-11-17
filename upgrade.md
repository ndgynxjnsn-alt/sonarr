# Upgrading from SonarQube 10.2 + PostgreSQL 16

**Target:** SonarQube Server **2026.5.5 Developer (LTA)** on **PostgreSQL 18.7**, the versions in this repo's `docker-compose.yml`.

> This guide is based on SonarSource / PostgreSQL documentation and has **not been tested end-to-end**.
> Rehearse it on a copy (restore a backup into a staging host) before doing it in production.

## 1. The path

SonarQube can't skip an LTA (Long-Term Active) release. Each LTA on the way must be installed at its **latest patch**.

```
SonarQube 10.2  ──►  2025.1.8 LTA  ──►  2026.1.8 LTA  ──►  2026.5.5 LTA
(PostgreSQL 16 stays during all SonarQube hops)               │
                                                              ▼
                                               PostgreSQL 16 ──► 18 (dump/restore)
```

| Hop | Image tag                       | Why this stop is required                                                   |
|-----|---------------------------------|-----------------------------------------------------------------------------|
| 1   | `sonarqube:2025.1.8-developer`  | First LTA after 10.x. 10.x → 2025.1 LTA is a direct update.                 |
| 2   | `sonarqube:2026.1.8-developer`  | 2025.1 LTA → 2026.1 LTA is direct. 2025.4 isn't a required stop.            |
| 3   | `sonarqube:2026.5.5-developer`  | "You must be on the 2026.1 LTA before updating to the 2026.5 LTA."          |
| 4   | `postgres:18.7`                 | Major Postgres versions can't read each other's data dir. Use dump/restore. |

Use the newest patch tag on Docker Hub at the time you upgrade. The tags above were current in November 2026.
The SonarSource **"Determine your update path"** calculator gives the authoritative path for your exact version.

**Why keep PostgreSQL at 16 until the end?**

* 16 is inside the supported range of every intermediate release: 2025.1, 2026.1 (14–18) and 2026.5 (15–18). SonarQube 10.2's official range ends at 15, but it is already running on 16.
* Official PostgreSQL 18 support only became solid with 2026.5. The community reported 2026.1 migration failures on PG 18.
* Changing one thing at a time makes rollbacks easier.

## 2. Breaking changes to check first

Read the release notes of every version you pass through. These are the ones most likely to affect this setup:

| Since         | Change                                                                                                                                                                                                                                                                                      |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 2025.1        | Product renamed to "SonarQube Server". Docker tags move from `10.x-developer` to `YYYY.N-developer`.                                                                                                                                                                                         |
| 2025.1        | Third-party plugins must support the new plugin API. Unmaintained ones (e.g. **`sonar-auth-oidc`**, tested only up to 9.9) can stop SonarQube from starting or break the login page. Remove them **before** hop 1 and switch to built-in SAML (see [keycloak.md](keycloak.md)). |
| 2026.4        | Scanners need **Java 21**. Java 17 scanner runtimes stop working.                                                                                                                                                                                                                           |
| 2026.4        | Elasticsearch 9 needs a writable `/tmp` in the container.                                                                                                                                                                                                                                   |
| 2026.5        | **PostgreSQL 15+ required**. PG 14 support removed.                                                                                                                                                                                                                                         |
| 2026.5        | Elasticsearch 8 → 9 triggers a **full reindex**. It is rebuilt into `data/es9`. Plan for extra startup time.                                                                                                                                                                                |
| 2026.5        | Default heap raised: web 1 GB, Compute Engine 1.5 GB. Setting `sonar.web.javaOpts` / `sonar.ce.javaOpts` **replaces** the defaults, so give the full flag set.                                                                                                                              |
| 2026.5        | Node.js 20 dropped for JS/TS analysis. It now uses an embedded Node 24.                                                                                                                                                                                                                     |
| 2026.5        | Security Hotspots are deprecated in favour of Security Issues. A migration is available from 2026.5.3 and will be forced in 2027.1.                                                                                                                                                         |
| 2026.5.5      | Webhook validation **blocks private-network destinations**. Internal Jenkins/GitLab webhooks can break. Check the release notes for the allow-list setting.                                                                                                                                  |
| every hop     | Issue backdating and rule changes can flip quality gates on "new code". Tell your teams.                                                                                                                                                                                                    |

## 3. Preparation (before hop 1)

1. **Downtime window.** Each hop runs a database migration and an Elasticsearch reindex. The time depends on DB size, from minutes to hours.
2. **Disk.** Keep DB disk usage **below 50 %**. Migrations can duplicate tables temporarily.
3. **Inventory plugins.** Run `docker compose exec sonarqube ls /opt/sonarqube/extensions/plugins`.
   Remove every plugin that has no 2025.1+ compatible release. Check them in *Administration → Marketplace* or on each plugin's GitHub page.
4. **Back up.** Back up both the database and the volumes:
   ```bash
   docker compose stop sonarqube
   docker compose exec -T postgres pg_dump -Fc -U sonarqube sonarqube > sonarqube-10.2-pg16.dump
   docker run --rm -v <project>_sonarqube_extensions:/src -v "$PWD":/dst alpine \
     tar czf /dst/sonarqube-extensions-10.2.tgz -C /src .
   ```
5. **Maintenance.** These commands lock tables, so run them during downtime:
   ```bash
   docker compose exec -T postgres psql -U sonarqube -d sonarqube \
     -c 'VACUUM FULL;' -c 'REINDEX DATABASE sonarqube;' -c 'ANALYZE;'
   ```
6. **Note the encoding.** The database must stay UTF-8 with a case-sensitive collation. Check with `\l` in psql.

## 4. SonarQube hops (repeat for each tag)

Move your old service definitions into this repo's `docker-compose.yml` layout first. Keep the old volume names, or point the `volumes:` entries at the existing ones with `external: true`.
During the hops, keep the old PostgreSQL 16 service and volume. The `postgres` service in this repo pins `postgres:18.7`, so temporarily set it to `postgres:16` with your existing PG 16 data volume.

For each `VERSION` in `2025.1.8`, `2026.1.8`, `2026.5.5`:

```bash
# 1. stop SonarQube (the DB keeps running)
docker compose stop sonarqube

# 2. switch the image
sed -i "s/^SONARQUBE_VERSION=.*/SONARQUBE_VERSION=$VERSION/" .env
docker compose pull sonarqube

# 3. start it and watch the logs
docker compose up -d sonarqube
docker compose logs -f sonarqube      # wait for "SonarQube is operational" or "DB migration required"

# 4. trigger the DB migration
#    Browser: https://sonarqube.org/setup  ->  "Upgrade"
#    or headless:
curl -sk -X POST https://sonarqube.org/api/system/migrate_db
curl -sk https://sonarqube.org/api/system/db_migration_status   # repeat until MIGRATION_SUCCEEDED

# 5. verify
curl -sk https://sonarqube.org/api/system/status                # {"status":"UP", "version":"<VERSION>..."}
```

After each hop:

* Log in and check *Administration → System* for warnings. Check that the background task queue drains.
* Take a fresh `pg_dump` before the next hop. That way a failed hop only costs one step.
* Don't keep the intermediate versions running long. The LTA stops are only migration waypoints.

If a hop fails, restore that hop's dump into an empty database, set the previous tag again and start over.
Never run an older SonarQube against a migrated schema.

**Elasticsearch.** Keep the `sonarqube_data` volume. SonarQube reindexes on its own when needed (e.g. into `data/es9` on 2026.5).
If startup hangs on index errors, stop SonarQube, delete `/opt/sonarqube/data/es*` in the data volume and start again. This forces a clean rebuild.

## 5. PostgreSQL 16 → 18

Do this after SonarQube runs 2026.5.x. The simplest reliable way is a logical dump and restore.

```bash
# 1. stop the writers
docker compose stop sonarqube pgbackup

# 2. final dump from PG 16 (custom format = parallel restore possible)
docker compose exec -T postgres pg_dump -Fc -U sonarqube sonarqube > sonarqube-2026.5-pg16.dump
docker compose stop postgres

# 3. switch to PG 18 on a NEW, empty volume (keep the PG 16 volume for rollback)
#    - image: postgres:18.7 (as in this repo)
#    - PG 18 images expect the volume at /var/lib/postgresql (NOT .../data);
#      data lives in /var/lib/postgresql/18/docker
#    - POSTGRES_DB / POSTGRES_USER / POSTGRES_PASSWORD as before
docker compose up -d postgres            # initdb creates an empty "sonarqube" DB

# 4. restore and refresh planner statistics
docker compose exec -T postgres pg_restore -U sonarqube -d sonarqube --no-owner --exit-on-error \
  < sonarqube-2026.5-pg16.dump
docker compose exec -T postgres vacuumdb -U sonarqube -d sonarqube --analyze-in-stages

# 5. bring everything back
docker compose up -d
curl -sk https://sonarqube.org/api/system/status
```

Notes:

* **Data checksums.** `initdb` in PG 18 enables them by default. That's harmless with dump/restore, but it is a reason `pg_upgrade` from a non-checksum PG 16 cluster needs `--no-data-checksums`.
* **Large databases.** For tens of GB, `pg_upgrade --link` (e.g. via the `tianon/postgres-upgrade:16-to-18` image) is much faster than dump/restore. It needs both data directories on the same filesystem and the checksum caveat above.
* **Backup image.** Switch `pgbackup` to the `:18` tag, as in this repo. The `pg_dump` major version must be ≥ the server's.
* **Rollback.** Remove the PG 18 volume and start the old PG 16 volume again. Nothing was written to it after step 2.

## 6. After the upgrade

1. Reanalyze the main branches of your important projects so new analyzers populate the data.
2. Update scanners: SonarScanner CLI / Maven / Gradle plugins on **Java 21** and JS/TS on Node 22+.
3. Run `VACUUM ANALYZE` again after a few days of normal load.
4. Hotspot migration, optional but forced in 2027.1. Preview it, then run it:
   ```bash
   curl -sk -u <admin-token>: -X POST 'https://sonarqube.org/api/hotspots/migrate_to_issues?dryRun=true'
   curl -sk -u <admin-token>: -X POST 'https://sonarqube.org/api/hotspots/migrate_to_issues'
   ```
5. Check that webhooks still fire (see the 2026.5.5 private-network restriction).
6. Check that the Alloy sidecar ships logs and metrics (`sonarqube_health_web_status` in Mimir). The monitoring passcode is `SONAR_WEB_SYSTEMPASSCODE`.
7. Delete the old PG 16 volume and intermediate dumps after a safe retention period.

## Sources

* Determine the update path (2026.5): https://docs.sonarsource.com/sonarqube-server/2026.5/server-update-and-maintenance/update/determine-path
* Pre-update steps: https://docs.sonarsource.com/sonarqube-server/server-update-and-maintenance/update/pre-update-steps
* Update procedure (Docker): https://docs.sonarsource.com/sonarqube-server/server-update-and-maintenance/update/update
* Post-update steps: https://docs.sonarsource.com/sonarqube-server/server-update-and-maintenance/update/post-update-steps
* LTA-to-LTA release notes 2026.5: https://docs.sonarsource.com/sonarqube-server/2026.5/server-update-and-maintenance/lta-to-lta-release-notes
* LTA-to-LTA release notes 2026.1: https://docs.sonarsource.com/sonarqube-server/2026.1/server-update-and-maintenance/lta-to-lta-release-notes
* 2026.5 LTA announcement: https://community.sonarsource.com/t/sonarqube-server-2026-5-lta/189075
* 9.9 → 2026.1 path discussion: https://community.sonarsource.com/t/upgrade-path-sonarqube-developer-9-9-0-enterprise-2026-1-lta-direct-or-staged/177248
* PG 18 migration issue on 26.1: https://community.sonarsource.com/t/sonarqube-community-edition-26-1-docker-image-with-posgresql-db-as-backend-fails-migration/172473
* Postgres Docker image (PG 18 volume layout): https://hub.docker.com/_/postgres
* PostgreSQL upgrade docs: https://www.postgresql.org/docs/18/upgrading.html
