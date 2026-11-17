# sonarr

Minimal SonarQube stack in one `docker-compose.yml`:

| Service     | Image                                      | Purpose                                         |
|-------------|--------------------------------------------|-------------------------------------------------|
| `traefik`   | `traefik:v3.7.19`                          | Reverse proxy, HTTP→HTTPS, Let's Encrypt (ACME) |
| `sonarqube` | `sonarqube:2026.5.5-developer`             | SonarQube Server Developer (2026.5 LTA)         |
| `postgres`  | `postgres:18.7`                            | Database                                        |
| `pgbackup`  | `prodrigestivill/postgres-backup-local:18` | Scheduled `pg_dump` with retention              |
| `alloy`     | `grafana/alloy:v1.22.0`                    | Sidecar: logs → Loki, metrics → Mimir           |

## Start

```bash
cp .env.example .env      # set POSTGRES_PASSWORD, SONAR_MONITORING_PASSCODE, ACME_EMAIL, SONAR_HOST
docker compose up -d
```

The host needs `vm.max_map_count >= 524288` (SonarQube's Elasticsearch needs it).
SonarQube is served at `https://${SONAR_HOST}` (default `sonarqube.org`). The first login is `admin` / `admin`, and SonarQube asks you to change it.

## TLS

Traefik's built-in ACME client requests and renews the certificate with an HTTP-01 challenge, so certbot isn't needed.
Port 80 must be reachable from the internet for that domain. Certificates are kept in the `letsencrypt` volume.

* Set `ACME_CA_SERVER=https://acme-staging-v02.api.letsencrypt.org/directory` for a dry run.
* Set `TLS_CERTRESOLVER=` (empty) for local or lab use. Traefik then serves its self-signed default certificate and never contacts Let's Encrypt.

## Backups

`pgbackup` writes gzipped plain-SQL dumps to `${BACKUP_DIR}` (default `./backups`) on `${BACKUP_SCHEDULE}` (default `@daily`).
It keeps 7 daily, 4 weekly and 6 monthly dumps.

```bash
docker compose exec pgbackup /backup.sh                    # back up now
docker compose stop sonarqube                               # restore:
gunzip -c backups/last/sonarqube-latest.sql.gz | \
  docker compose exec -T postgres psql -U sonarqube -d sonarqube
docker compose start sonarqube
```

Restore into an empty database. Drop and recreate it first, or restore into a fresh `postgres_data` volume.

## Observability (Grafana Alloy sidecar)

Alloy's config is in [`alloy/config.alloy`](alloy/config.alloy). Alloy's UI is on container port 12345, which isn't published.

* **Logs**: Alloy reads the Docker logs of every container in compose project `sonarr` and pushes them to Loki (`LOKI_URL`) with the labels `service_name`, `container` and `project="sonarr"`.
* **Metrics**: Alloy pushes these to Mimir (`MIMIR_URL`) with Prometheus remote write. Every series gets `project="sonarr"`.
  * `job="sonarr/traefik"`: Traefik's Prometheus metrics, served on the internal entrypoint `:8082`.
  * `job="sonarr/sonarqube"`: `/api/monitoring/metrics`, authenticated with `SONAR_MONITORING_PASSCODE`. SonarQube receives it as `SONAR_WEB_SYSTEMPASSCODE`.
  * `job="sonarr/postgres"`: Alloy's built-in postgres exporter.

The default URLs point at Loki (`:3100`) and Mimir (`:9009`) published on the Docker host. Change them in `.env` for another LGTM stack.

Example queries: `{project="sonarr", service_name="sonarqube"}` in Loki, and `sonarqube_health_web_status` or `pg_up{job="sonarr/postgres"}` in Mimir.

## Further guides

* [upgrade.md](upgrade.md): upgrading from PostgreSQL 16 and SonarQube 10.2.
* [keycloak.md](keycloak.md): recovering when an auth plugin such as `sonar-auth-oidc` breaks the login screen.
