# Logbox

Logbox is a lightweight observability sandbox that ships a pre-wired Grafana, Loki, and Promtail stack so you can explore log aggregation, dashboards, and alerting with almost no setup. Synthetic log generators keep the demo lively while you experiment with LogQL and tailor the scrape pipeline to match your own services.

## Features at a Glance
- Loki 2.9 with a minimal filesystem-backed config for local experiments.
- Promtail scraping Docker JSON logs, Linux-style syslog files, and sample app JSON logs (including the `svc` label used throughout the dashboards).
- Grafana 11 with provisioning for datasources, dashboards, and unified alerting policies.
- Synthetic log producers (`loggen-app`, `loggen-auth`) so panels and alerts have immediate data.

## Prerequisites
- Docker Engine 20.10+ and Docker Compose v2.
- macOS or Linux host (paths in `docker-compose.yml` assume a Unix-like system).

## Quick Start
1. Clone the repo and switch into it.
2. Launch the stack:
   ```bash
   docker compose up -d
   ```
3. Confirm the containers are healthy:
   ```bash
   docker compose ps
   ```
4. Open Grafana at [http://localhost:3000](http://localhost:3000). Default credentials are `admin` / `admin` (you will be prompted to change the password on first login).
5. Navigate to *Dashboards → Application & System Errors (Slim)* to see the pre-built view fed by the synthetic logs.

### Smoke-Check Loki Directly
To verify Loki is receiving data with the expected labels:
```bash
curl -s --get http://localhost:3100/loki/api/v1/query \
  --data-urlencode 'query=sum by (svc)(count_over_time({job="app", level="ERROR"}[5m]))'
```
You should see a series for `svc="my-dotnet-api"` with non-zero counts.

## What Runs Where
- **loki** – Uses `loki/loki-config.yml`, stores data in the `loki_data` volume, and exposes port `3100` on the host for ad-hoc queries.
- **promtail** – Loads `promtail/promtail-config.yml`. Key scrape jobs:
  - `syslog`: `/var/log/*.log` on the host (adjust paths for your environment).
  - `docker`: JSON logs for any container on the same host.
  - `dev_app_json`: `/logs/*.log` mounted from `./dev-logs`; parses JSON and promotes `level`, `svc` labels.
- **grafana** – Provisions datasources, dashboards, and alerting assets from `grafana/provisioning/**`. Credentials come from environment variables in `docker-compose.yml`.
- **loggen-app** – Writes rotating JSON error events into `./dev-logs/app.log`, providing data for the application dashboard.
- **loggen-auth** – Streams failed SSH-like messages to exercise the auth alert rule.

## Dashboards & Alerting
- `grafana/provisioning/dashboards/app-errors.json` visualizes error rates, counts, top messages, and recent error logs filtered by `svc` and the `level="ERROR"` label.
- Alerting assets in `grafana/provisioning/alerting/` demonstrate Grafana Unified Alerting:
  - `contact-points.yml` defines placeholder email and Slack receivers.
  - `notification-policies.yml` routes high-severity alerts to the Slack contact point.
  - `rules.yml` contains example Loki-based alerts (failed logins, application error spikes). Update label matchers or thresholds to mirror your environment before relying on them.

## Customizing Promtail
- **Change log locations:** Edit the `__path__` entries in `promtail/promtail-config.yml` to point to your application logs. Mount additional host paths into the Promtail container as needed.
- **Adjust labels:** Ensure the labels you reference in Grafana (e.g., `svc`, `level`) are extracted in the Promtail pipeline using the `json` and `labels` stages. The provided config already emits `svc` from the synthetic logs.
- **Add pipelines:** Copy the pattern used in `dev_app_json` to parse other JSON fields (such as `status`, `user`, `elapsedMs`).

## Repository Layout
```
docker-compose.yml              # Defines Loki, Promtail, Grafana, and log generators
grafana/provisioning/           # Datasources, dashboards, unified alerting assets
loki/loki-config.yml            # Minimal single-node Loki configuration
promtail/promtail-config.yml    # Promtail scrape jobs and pipelines
dev-logs/app.log                # Sample application log file mounted into Promtail/loggen-app
```

## Helpful Commands
- Rebuild or pull the latest images:
  ```bash
  docker compose pull
  docker compose up -d --build
  ```
- Tail Promtail logs while debugging scrape issues:
  ```bash
  docker compose logs -f promtail
  ```
- Tear everything down (containers only):
  ```bash
  docker compose down
  ```
- Tear down and remove volumes (destroys stored logs and Grafana state):
  ```bash
  docker compose down -v
  ```

## Troubleshooting
- **Dashboard shows no data:** Confirm Promtail is emitting the `svc` and `level` labels and that Grafana queries filter on labels instead of raw message content.
- **Permission errors reading host logs:** Adjust the volume mounts and file permissions so the Docker user (typically UID 0 in the container) can read your log files.
- **Alert delivery fails:** Replace the placeholder email address and Slack webhook in `grafana/provisioning/alerting/contact-points.yml`, then restart Grafana to apply changes.

Happy logging!
