# PostgreSQL monitoring stack (postgres_exporter + Prometheus + Grafana)

## 1. Create a monitoring user in PostgreSQL

On the PostgreSQL server you want to monitor (primary), run these queries:

```sql
CREATE USER postgres_exporter WITH PASSWORD 'change-me-strong-password' CONNECTION LIMIT 5;
ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter, pg_catalog;
GRANT pg_monitor TO postgres_exporter;
GRANT SELECT ON pg_stat_replication TO postgres_exporter;
GRANT SELECT ON pg_replication_slots TO postgres_exporter;
```

If your server doesn't allow connections from outside (from inside Docker), you need to add/check a line in `pg_hba.conf` for the Docker network's IP range (usually `172.16.0.0/12` by default, or whatever network compose creates) and `listen_addresses = '*'` in `postgresql.conf`, then reload PostgreSQL.

## 2. Create the `.env` file

```bash
cp .env.example .env
```

Fill in the values for `PG_HOST`, `PG_USER`, `PG_PASSWORD`, and the Grafana admin password.

- If PostgreSQL runs on the host itself (not inside Docker): `PG_HOST=host.docker.internal` (Linux needs the `extra_hosts` setting, explained below)
- If PostgreSQL is also inside a separate container on the same Docker network: `PG_HOST=<postgres service name>`

### Linux note for host.docker.internal
If you're on Linux (not Docker Desktop on Mac/Windows), add this to the `postgres-exporter` service in `docker-compose.yml`:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

## 3. Bring the stack up

```bash
docker compose up -d
```

## 4. Access

| Service | Address | Default user/password |
|---|---|---|
| Grafana | http://localhost:3000 | Whatever you set in `.env` |
| Prometheus | http://localhost:9090 | - |
| Alertmanager | http://localhost:9093 | - |
| postgres_exporter raw metrics | http://localhost:9187/metrics | - |

## 5. Checking the setup is healthy

1. Go to `http://localhost:9090/targets` and make sure the target named `postgres` has status **UP**.
2. If it's not UP, check the exporter logs: `docker compose logs postgres-exporter`
3. Log into Grafana — the **PostgreSQL / PostgreSQL - Replication & Health** dashboard is already provisioned automatically (no manual import needed).

## 6. Monitoring the replica separately too (optional)

If you have multiple standbys and want to see status from their own point of view too (e.g. `pg_stat_wal_receiver`), add another exporter service in `docker-compose.yml` (a copy of `postgres-exporter` with a new name and a `DATA_SOURCE_URI` pointing at that replica's IP), and add its new target in `prometheus/prometheus.yml` with a different `label: instance` (an example is commented in the file).

## 7. A more general-purpose dashboard (optional)

The dashboard built into this project focuses only on replication and overall health. If you want a much more comprehensive dashboard with dozens of panels (query performance, locks, vacuum, etc.), go into Grafana to Dashboards → New → Import and enter code `9628` (the community dashboard for this exporter).

## 8. Alerts

Rules ready in `prometheus/alert_rules.yml`:
- Lag greater than 30 seconds → warning
- Lag greater than 120 seconds → critical
- No replica connected → critical
- Replication slot inactive (fills up primary's disk) → warning
- PostgreSQL itself down → critical

To receive these alerts in Slack/Email, fill in the `receivers` section in `prometheus/alertmanager.yml`.

---

**Sources used:** the official `prometheus-community/postgres_exporter` repo on GitHub (for the image name and env vars), Grafana Labs documentation for dashboard 9628, and postgresql.org documentation on `pg_stat_replication` / `pg_stat_wal_receiver`.
