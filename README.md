# sonarr

Minimal SonarQube stack in one `docker-compose.yml`:

| Service     | Image                                      | Purpose                                         |
|-------------|--------------------------------------------|-------------------------------------------------|
| `traefik`   | `traefik:v3.7.19`                          | Reverse proxy, HTTP→HTTPS, company ACME (EAB)   |
| `sonarqube` | `sonarqube:2026.5.5-developer`             | SonarQube Server Developer (2026.5 LTA)         |
| `postgres`  | `postgres:18.7`                            | Database                                        |
| `pgbackup`  | `prodrigestivill/postgres-backup-local:18` | Scheduled `pg_dump` with retention              |
| `alloy`     | `grafana/alloy:v1.22.0`                    | Sidecar: SonarQube + host telemetry → Loki/Mimir |

## Start

```bash
cp .env.example .env      # set passwords, SONAR_HOST and the ACME_* values
docker compose up -d
```

The host needs `vm.max_map_count >= 524288` (SonarQube's Elasticsearch needs it).
SonarQube is served at `https://${SONAR_HOST}` (default `sonarqube.org`). The first login is `admin` / `admin`, and SonarQube asks you to change it.

## TLS

Certificates come from the **company ACME server**. Traefik's built-in ACME client (lego) does issuance and renewal, so certbot isn't needed.
It uses External Account Binding (EAB) and the HTTP-01 challenge. The ACME server must be able to reach port 80 for `SONAR_HOST`.

| Variable               | Meaning                                                                                  |
|------------------------|------------------------------------------------------------------------------------------|
| `ACME_CA_SERVER`       | Directory URL, e.g. `https://acme.company.example/acme/directory`                        |
| `ACME_EAB_KID`         | EAB key identifier from the CA                                                           |
| `ACME_EAB_HMAC`        | EAB HMAC key, base64url-encoded (as handed out by the CA)                                |
| `ACME_EMAIL`           | Account contact address                                                                  |
| `ACME_KEY_TYPE`        | Certificate key type, default `EC256` (`RSA2048`, `RSA4096`, … if the CA requires RSA)   |
| `ACME_CA_CERTIFICATES` | Optional. Path of the company root CA PEM (put it in `./certs/`, e.g. `/certs/company-root-ca.pem`) if the ACME endpoint's own TLS certificate isn't publicly trusted |
| `TLS_CERTRESOLVER`     | `company` (default). Leave it empty to skip ACME and serve Traefik's self-signed certificate (lab use) |

Certificates and the ACME account are stored in the `acme` volume (`/acme/acme.json`).
If the CA wants DNS-01 or TLS-ALPN-01 instead of HTTP-01, replace the `httpchallenge` flag in `docker-compose.yml`.
The options are `--certificatesresolvers.company.acme.tlschallenge=true`, or `...dnschallenge.provider=<provider>` plus that provider's env vars.

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

[`alloy/config.alloy`](alloy/config.alloy) watches only SonarQube and the host:

* **Logs → Loki** (`LOKI_URL`): SonarQube's log files (`sonar`, `web`, `ce`, `es`, `access`). Alloy reads them from the `sonarqube_logs` volume, mounted read-only. Labels: `service_name="sonarqube"`, `filename`.
* **Metrics → Mimir** (`MIMIR_URL`), all with `project="sonarr"`:
  * `job="sonarqube"`: `/api/monitoring/metrics`, authenticated with `SONAR_MONITORING_PASSCODE`. SonarQube receives it as `SONAR_WEB_SYSTEMPASSCODE`.
  * `job="integrations/unix"`: host metrics equivalent to node_exporter. They are read from `/proc`, `/sys` and `/` mounted read-only, with `pid: host`.

The default URLs point at Loki (`:3100`) and Mimir (`:9009`) published on the Docker host. Change them in `.env` for another LGTM stack.

Example queries: `{service_name="sonarqube", filename=~".*/web.log"}` in Loki, and `sonarqube_health_web_status` or `node_load1{project="sonarr"}` in Mimir.
Network-interface metrics show the container's own interfaces, not the host's. Add `network_mode: host` to `alloy` if you need host NIC stats. It then reaches SonarQube via the published HTTPS URL instead of `sonarqube:9000`.

## Further guides

* [upgrade.md](upgrade.md): upgrading from PostgreSQL 16 and SonarQube 10.2.
* [keycloak.md](keycloak.md): recovering when an auth plugin such as `sonar-auth-oidc` breaks the login screen.
