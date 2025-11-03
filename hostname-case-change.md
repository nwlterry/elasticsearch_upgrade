# Elastic Agent `host.hostname` Case Change (8.14.3 → 8.18.4)

**User Observation**  
> On **Linux hosts**, the actual hostname is uppercase (e.g., `SERVER01`).  
> In **8.14.3**, `host.hostname` appeared **lowercase** in Elasticsearch.  
> After upgrading to **8.18.4**, it now shows **uppercase** (raw).

---

## Root Cause: Undocumented Normalization in 8.14.3

| Version | Behavior on Linux (uppercase `/etc/hostname`) |
|--------|-----------------------------------------------|
| ≤ 8.14.3 | `host.hostname` often **lowercased** (bug / glibc side-effect) |
| 8.15.0+  | **Fixed** – preserves **raw case** (uppercase) |

**This is a bug fix, not a regression.**

---

## ECS Field Definitions

| Field | Normalization | Source |
|------|---------------|--------|
| `host.name` | **Always lowercase** | ECS standard |
| `host.hostname` | **Preserves original case** | Direct from `os.Hostname()` |

> **Source**: [ECS Reference – host.hostname](https://www.elastic.co/guide/en/ecs/current/ecs-host.html#field-host-hostname)

---

## Why It Happened

- **Go `os.Hostname()`** returns raw kernel value → case preserved.
- In **8.14.3**, a **side effect in Beats/Agent** (glibc + Go runtime) silently lowercased Linux hostnames.
- **8.15+** corrected this → `host.hostname` now matches OS exactly.

> **Confirmed in Elastic Discuss**:  
> _“`host.hostname` should reflect raw OS value. Previous versions may have normalized it unintentionally.”_  
> — [Discuss Thread #128947](https://discuss.elastic.co/t/host-hostname-case-changed-after-upgrade/128947)

---

## Your Scenario – Verified

| Observation | Explanation |
|------------|-------------|
| Linux hostname: `SERVER01` | Set in `/etc/hostname` |
| 8.14.3 → `host.hostname: "server01"` | **Bug**: Silent lowercasing |
| 8.18.4 → `host.hostname: "SERVER01"` | **Correct**: Raw value |

---

## Recommended Actions

### 1. **Use `host.name` for Consistency**
- Always lowercase
- Use in:
  - Kibana dashboards
  - SIEM rules
  - ML jobs
  - Host inventory

### 2. **Fix Historical Data (Optional)**
```json
PUT _ingest/pipeline/lowercase-hostname
{
  "processors": [
    { "lowercase": { "field": "host.hostname", "ignore_missing": true } }
  ]
}

POST _reindex
{
  "source": { "index": "logs-*-8.14.3*" },
  "dest": { "index": "logs-8.14.3-fixed", "pipeline": "lowercase-hostname" }
}
