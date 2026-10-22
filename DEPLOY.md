# NodX Colo Operator Pilot — Deployment Guide

> **Target:** Local pilot evaluation in 30 minutes.
> **Production path:** Same configs, same agent, scale horizontally.

---

## Prerequisites

| Requirement | Minimum |
|-------------|---------|
| Docker | 24.x |
| Docker Compose | v2.x (plugin) |
| RAM | 4 GB available |
| CPU | 2 cores |
| Disk | 10 GB free |
| Ports | 3000, 3001, 5432, 6379 free |

**OS:** Linux, macOS, Windows (WSL2)

---

## Quick Start (5 minutes)

```bash
# 1. Clone
git clone https://github.com/nodx-ops/pilot-colo-operator.git
cd pilot-colo-operator

# 2. Start stack
docker compose up -d

# 3. Verify health
docker compose ps
# All services should show "healthy"

# 4. Open UI
open http://localhost:3000
# Login: admin@nodx-ops.com / nodxpilot (change in .env)

# 5. Import dashboards
# Grafana UI → Dashboards → Import → select each dashboards/*.json

# 6. Load sample data
docker compose exec api nodx import --dry-run import/sample-racks.csv
# Review output → if clean:
docker compose exec api nodx import import/sample-racks.csv

# 7. Verify data appears
# Refresh UI → Fleet Overview should show 50 racks
```

---

## Configure for Your Gear

### 1. Edit `nodx-agent.yaml`

```yaml
redfish:
  targets:
    - host: "bmc-rack-01"        # Your BMC hostname/IP
      username: "root"           # BMC username
      password: "${REDFISH_PASSWORD}"  # Set in .env
      port: 443
      verify_ssl: false          # Set true for production

snmp:
  targets:
    - host: "pdu-rack-01"        # Your PDU hostname/IP
      community: "public"        # Your SNMP community
      version: "2c"              # or "3"
      port: 161

mqtt:
  broker: "mqtt://sensor-gateway:1883"
  # username/password in .env
```

### 2. Create `.env` file

```bash
cat > .env <<'EOF'
POSTGRES_PASSWORD=changeme
AUTH_JWT_SECRET=changeme-32-chars-min
ADMIN_PASSWORD=changeme
AGENT_SECRET=changeme
REDFISH_PASSWORD=your-bmc-password
MQTT_USERNAME=your-mqtt-user
MQTT_PASSWORD=your-mqtt-pass
EOF
```

### 3. Restart agent

```bash
docker compose restart agent
```

---

## Import Your Rack Data

### From CSV (recommended)

```bash
# Dry-run first (safe preview)
docker compose exec api nodx import --dry-run your-racks.csv

# If clean:
docker compose exec api nodx import your-racks.csv
```

**CSV columns required:**
```csv
rack_id,site_id,name,u_height,power_capacity_watts,location_row,location_rack
R01,SITE-A,Rack-01,42,20000,A,01
```

### From NetBox

```bash
# Export from NetBox → JSON → Import
docker compose exec api nodx import --dry-run import/netbox-export.json
docker compose exec api nodx import import/netbox-export.json
```

---

## Dashboards

Import each JSON in `dashboards/` to Grafana:

| Dashboard | Purpose |
|-----------|---------|
| `fleet-overview.json` | Executive view: health, PUE, alerts, stranded power |
| `rack-detail.json` | Per-rack: power, thermal, inventory, alerts |
| `capacity-planning.json` | Stranded power, node-fit, headroom, forecast |
| `sustainability.json` | PUE trend, CO2e, reclaimed power, efficiency score |

**Import:** Grafana → Dashboards → Import → Upload JSON

---

## Alert Rules

Rules in `alert-rules/` are auto-loaded by the API on startup.

| File | Coverage |
|------|----------|
| `threshold-critical.yaml` | Temp > 35°C, Power > 90%, Disk > 90%, UPS battery, phase imbalance |
| `anomaly-detection.yaml` | Z-score > 3σ on power, thermal, network (24 hourly baselines) |

**Test an alert:**
```bash
# Inject synthetic anomaly
docker compose exec api nodx test anomaly --rack R01 --metric temp --value 42
# Watch alert fire in UI → Alerts page
```

---

## Webhook Integration

Alerts fire → webhook. Configure in UI:

| Destination | Template |
|-------------|----------|
| Slack | `#alerts` channel, mentions on-call |
| PagerDuty | Critical → P1, Warning → P3 |
| Microsoft Teams | Channel with action buttons |
| Custom HTTP | Your ticketing system (Jira, ServiceNow) |

**HMAC verification:** Each webhook includes `X-NodX-Signature` (SHA256).

---

## Scaling to Production

| Pilot | Production |
|-------|------------|
| `docker compose` | Kubernetes (Helm chart) or VM fleet |
| Single API | API replicas + load balancer |
| Embedded Postgres | Managed Postgres (RDS, CloudSQL) |
| Embedded Redis | Redis Cluster / ElastiCache |
| Local agent | DaemonSet / systemd on each host |
| Self-signed certs | Let's Encrypt / ACME / cert-manager |

**Config unchanged.** Same `nodx-agent.yaml`, same dashboards, same alert rules.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Agent won't register | `docker compose logs agent` — verify `NODX_AGENT_SECRET` matches API |
| No metrics | `docker compose logs agent` — check Redfish/SNMP/MQTT connectivity |
| UI blank | `docker compose logs ui` — verify `API_BASE_URL` |
| Import fails | `docker compose exec api nodx import --dry-run ...` — fix CSV/JSON |
| Alerts not firing | Verify Prometheus scrape targets up; check rule syntax |

**Logs:**
```bash
docker compose logs -f api      # API + ingestion
docker compose logs -f agent    # Agent collection
docker compose logs -f ingestion # Queue processor
```

---

## Security Notes

- **Change all defaults** in `.env` before exposing
- **Use secrets manager** in production (Vault, AWS Secrets Manager)
- **TLS everywhere** — terminate at load balancer, mTLS between services
- **Network policies** — restrict inter-service traffic
- **Audit logs** — enable in API config

---

## Support

- **Documentation:** https://docs.nodx-ops.com
- **Issues:** https://github.com/nodx-ops/pilot-colo-operator/issues
- **Contact:** https://nodx-ops.com/contact/

---

## License

Apache 2.0 — use freely, modify, distribute.