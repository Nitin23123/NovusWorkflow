# New Database Spec — FastAPI backend (Novus Aegis)

**For:** DB engineer · **Goal:** a fresh Postgres database that works with the new FastAPI (`dev`) backend, holding the same data as `deception_db`.

---

## TL;DR

1. Give me a **fresh, empty Postgres 14+ database** + a role.
2. The FastAPI app **creates the entire schema itself on boot** (runs its migrations — all tables, indexes, RLS, a default org, and demo data). So an **empty DB + credentials is enough** to get a fully working app.
3. If we want `deception_db`'s **real data**, copy it in *after* the app builds the schema (mapping notes in §5). It's a **guided migration, not a plain dump/restore**, because the new schema is newer than `deception_db` (that's exactly why the old DB doesn't work with the new code).

---

## 1. Server / role requirements

- **Postgres 14+**
- **Extension:** `pgcrypto` — the migrator runs `CREATE EXTENSION IF NOT EXISTS pgcrypto;` (used for `gen_random_uuid()` + hashing). The role needs rights to create it, or pre-create it.
- **SSL:** please tell me if SSL is required to connect (yes/no).
- **Role:** one role that **owns the database** and can create tables/extensions. Simplest option, fine for dev.
- **Optional (RLS):** if we want row-level tenant security enforced at runtime, also create a second **`NOBYPASSRLS`** app role (username + password) and send it. Otherwise the single owner role is fine.

## 2. How the schema is created (you don't write any DDL)

On startup the app runs these migrations, idempotently, in order:

```
001_base_schema      005_tenancy         008_automations_integrations
002_ingestion_module 006_rls             500–530 (demo seed data)
003_threat_indicators 007_recon_threat   600_auth
                                          700_agents · 701_indicator_flags
```

They build everything in §3, add a **default organization**, and seed a demo dataset. Files live in the repo at `migrations/`.

## 3. Tables the database will have (~39)

**Identity & tenancy**
| Table | Key columns |
|---|---|
| `organizations` | id (uuid), name, slug, plan, status, timestamps |
| `users` | id (**text**), email, full_name, role, mfa_enabled, status, password_hash, last_login |
| `agents` | id (uuid), name, type, enabled, instances, status, config (jsonb) |
| `audit_logs` | id, user_id, action, target, details (jsonb), ip_address, created_at |

**Alerts & investigations**
| Table | Key columns |
|---|---|
| `alerts` | id (**text**), title, description, severity, status, source, ai_summary, conclusion, confidence, mitre_tactic/technique |
| `alert_entities` | alert_id, entity_type, entity_value |
| `investigations` | id (**text**), title, priority, status, progress, summary, assigned_to, started_at, closed_at |
| `investigation_findings` / `_notes` / `_alerts` | investigation_id + finding/note/alert link |
| `remediation_tasks` | investigation_id, task_description, status, assigned_to, due_at |
| `evidence` | investigation_id, evidence_type, file_name, sha256_hash |

**Attackers, honeypots & simulations (decoy)**
| Table | Key columns |
|---|---|
| `attackers` | id (text), ip_address, actor_alias, country, reputation_score, total_sessions, first/last_seen |
| `honeypot_templates` | id, name, service_type, vulnerability_profile (jsonb) |
| `honeypot_instances` | id, template_id, status, public_ip, region, cloud_provider, ttl_minutes |
| `attack_sessions` | id, attacker_id, honeypot_id, protocol, started_at, ended_at, ai_summary |
| `attack_commands` | session_id, command, output, suspicious, executed_at |
| `honeytokens` | id (uuid), token_type, location, triggered, source_ip |
| `simulations` / `simulation_runs` | name, attack_type, parameters/results (jsonb), status, progress |

**Threats & ingestion**
| Table | Key columns |
|---|---|
| `threat_events` | id (uuid), tenant_id, source, event_type, ioc_type, ioc_value, confidence, severity, tags, raw_event (jsonb) |
| `threat_indicators` | id (uuid), indicator_type, value, confidence, tags, metadata, **watchlisted**, **false_positive** |
| `ingest_sources` / `ingest_dead_letters` | adapter config + failed-payload dead-letter queue |
| `threat_actors` / `threat_iocs` / `threat_feeds` | org_id, ext_id, **payload (jsonb)** — stores original JSON, keyed (org_id, ext_id) |

**Recon**
| Table | Key columns |
|---|---|
| `recon_assets` / `recon_scans` | org_id, ext_id, **payload (jsonb)**, created_at |

**Policies & automations**
| Table | Key columns |
|---|---|
| `response_policies` | id (uuid), name, status, trigger, action, requires_approval, conditions/actions (jsonb) |
| `automation_rules` | name, trigger_type/condition, priority, enabled, actions (jsonb), org_id |
| `playbooks` / `playbook_runs` | steps/result (jsonb), status, progress, org_id |
| `integrations` | name, type, status, api_endpoint, auth_method, config (jsonb), org_id |

**AI copilot & system**
| Table | Key columns |
|---|---|
| `ai_threads` / `ai_queries` | thread title/context; query, response, llm_model, tokens_used |
| `system_metrics` | cpu/memory/network usage, active_sessions, recorded_at |
| `synthetic_clean_logs` | src_ip, country, threat_type + behaviour flags |

## 4. Multi-tenancy — the key difference from `deception_db`

- There is a `organizations` table and a **default org: `00000000-0000-0000-0000-000000000001`**.
- **Every business table has an `org_id UUID NOT NULL DEFAULT '…001'`** referencing `organizations(id)`.
- **Row-Level Security (RLS)** policies enforce per-org isolation at query time.

👉 `deception_db` predates all of this (no `org_id`, no RLS, older columns like a missing `ioc_value`) — which is why the new code errors against it (`column "org_id" does not exist`). The new DB fixes that.

## 5. Data — "same data as `deception_db`"

**Option A — fresh demo data (zero effort).** The app auto-seeds a small demo dataset (1 row per core table: a user, alert, investigation, attacker, honeypot, session, policy, threat indicators, plus recon/threat/automation samples). Good enough to see the app fully working immediately.

**Option B — carry over `deception_db`'s real data.** Recommended sequence:

1. **Boot the app once against the empty new DB** → it creates the full new schema + default org.
2. **Copy table-by-table** from `deception_db` into the new DB, tagging every row with the default org:
   ```sql
   INSERT INTO <table> (<matching columns>, org_id)
   SELECT <matching columns>, '00000000-0000-0000-0000-000000000001'
   FROM deception_db.<table>;
   ```
3. **Mind the gaps** (new schema is newer than `deception_db`):
   - **New tables** with no source data → leave seeded/empty: `organizations`, `agents`.
   - **New columns** → let them take their defaults: `org_id` (all tables), `threat_indicators.watchlisted` / `false_positive`, any `threat_events` columns missing in the old DB (e.g. `ioc_value`).
   - **JSON-payload tables** (`threat_actors`, `threat_iocs`, `threat_feeds`, `recon_assets`, `recon_scans`): store each old record's JSON into `payload`, key by `(org_id, ext_id)`.
   - **IDs:** core business tables use human-readable **TEXT** ids (`ALT-001`, `INV-2024-001`); newer tables use **UUID**. Copy ids as-is.

> Honest note: because `deception_db` is on an older schema, Option B is a **guided migration**, not a clean `pg_dump`/restore — a few tables/columns won't line up 1:1 and need the defaults above. If the goal is just "a working app with realistic data," **Option A is the fast path**; use **Option B** only if we specifically need the historical records preserved.

## 6. Please hand back

- **host, port, database name, username, password**
- **SSL required?** (yes/no)
- *(if you seed a non-default organization)* its **org UUID**

Once I have these, I point the app's `.env` at the new DB, restart, and it builds/seeds itself — working within a minute.
