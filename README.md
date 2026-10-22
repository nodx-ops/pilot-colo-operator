# NodX Colo Operator Pilot

> **50-rack colocation monitoring — running in 30 minutes.**

Pre-tuned for Dell, HPE, Supermicro servers (Redfish) + APC, Vertiv PDUs (SNMP) + environmental sensors (MQTT). Includes adaptive anomaly detection, stranded-power recovery, and 4 Grafana dashboards.

## Quick Start

```bash
# 1. Clone
git clone https://github.com/nodx-ops/pilot-colo-operator.git
cd pilot-colo-operator

# 2. Start NodX stack (API + UI + DB)
docker compose up -d

# 3. Open dashboard
open http://localhost:3000

# 4. Import Grafana dashboards
# Settings → Dashboards → Import → select dashboards/*.json

# 5. Load sample data (dry-run first)
docker compose exec api nodx import --dry-run import/sample-racks.csv
# Review output → approve
docker compose exec api nodx import import/sample-racks.csv
```

## What's Included

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Local NodX stack (Postgres, Redis, API, Ingestion, UI) |
| `nodx-agent.yaml` | Agent config for colo gear (Redfish, SNMP, MQTT) |
| `dashboards/*.json` | 4 Grafana dashboards (fleet, rack, capacity, sustainability) |
| `alert-rules/*.yaml` | Production alert rules (threshold, anomaly, capacity, hardware) |
| `import/sample-racks.csv` | 50-rack sample data for instant dashboards |
| `import/netbox-export.json` | NetBox → NodX import example |
| `DEPLOY.md` | Detailed deployment guide |

## Requirements

- Docker + Docker Compose v2
- 4 GB RAM, 2 CPU cores
- Ports 3000 (UI), 3001 (API), 5432 (Postgres), 6379 (Redis) free

## For Your Real Gear

Edit `nodx-agent.yaml`:
```yaml
redfish:
  targets:
    - host: "bmc-rack-01"
      username: "root"
      password: "${REDFISH_PASSWORD}"
snmp:
  targets:
    - host: "pdu-rack-01"
      community: "public"
      version: "2c"
mqtt:
  broker: "mqtt://sensor-gateway:1883"
```

Then restart agent: `docker compose restart agent`

## Pilot → Production

Same configs, same agent, same dashboards. Pilot → production is a scale change, not a rewrite.

## Support

- Documentation: https://docs.nodx-ops.com
- Issues: [GitHub Issues](https://github.com/nodx-ops/pilot-colo-operator/issues)
- Contact: https://nodx-ops.com/contact/

## License

Apache 2.0 — use freely, modify, distribute.