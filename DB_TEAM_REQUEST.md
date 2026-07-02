# Request to DB Team — Tables needed to unblock remaining APIs

**From:** Backend / API
**Re:** Novus platform — 8 missing tables blocking 5 feature areas

## TL;DR

We just shipped ~45 new API endpoints that only use tables **already in the
database** (alerts, investigations, attack_sessions, threat_indicators,
honeypot_instances, users, user_sessions, ai_threads/ai_queries, audit_logs).
Those are done and live.

The features below are **fully built on the frontend and the API routes are
either ready or trivial to add** — they are blocked **only** because the
underlying tables don't exist yet. As soon as each table is created (and
lightly seeded), the corresponding feature lights up with no further backend
work in most cases.

Please create the following 8 tables. DDL is provided and ready to run.

---

## Priority order

| # | Table(s) | Unblocks | API status |
|---|----------|----------|------------|
| 1 | `simulations`, `simulation_runs` | Decoy Center → Threat Simulation tab | Routes READY |
| 2 | `honeytokens` | Decoy Center → Honeytokens | Routes READY |
| 3 | `response_policies` | Response Policies page (create/edit/delete/toggle/dry-run) | Routes READY once table exists |
| 4 | `automation_rules`, `playbooks`, `playbook_runs` | Automations page | Routes to be added |
| 5 | `integrations` | Integrations page | Routes to be added |

---

## DDL — ready to run

### 1. `response_policies` — Response Policies page
Currently falls back to `server/data/response_policies.json` (read-only).

```sql
CREATE TABLE response_policies (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              TEXT NOT NULL,
    description       TEXT,
    status            TEXT DEFAULT 'active',          -- active, inactive
    trigger           TEXT,                           -- e.g. "alert.severity == critical"
    action            TEXT,                           -- e.g. "auto_isolate", "block_ip"
    requires_approval BOOLEAN DEFAULT false,
    conditions        JSONB DEFAULT '[]',
    actions           JSONB DEFAULT '[]',
    trigger_count     INTEGER DEFAULT 0,
    last_triggered    TIMESTAMP,
    created_at        TIMESTAMP DEFAULT NOW(),
    updated_at        TIMESTAMP DEFAULT NOW()
);
```

### 2. `simulations` — Decoy Center → Threat Simulation
```sql
CREATE TABLE simulations (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              TEXT NOT NULL,
    description       TEXT,
    attack_type       TEXT,                           -- brute_force, lateral_movement, ...
    parameters        JSONB DEFAULT '{}',
    target_honeypots  JSONB DEFAULT '[]',             -- array of honeypot IDs
    detection_checks  JSONB DEFAULT '[]',
    expected_outcomes JSONB DEFAULT '[]',
    created_at        TIMESTAMP DEFAULT NOW()
);
```

### 3. `simulation_runs` — run history + logs
```sql
CREATE TABLE simulation_runs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    simulation_id     UUID REFERENCES simulations(id),
    status            TEXT DEFAULT 'running',         -- running, completed, failed
    progress          INTEGER DEFAULT 0,              -- 0-100
    started_at        TIMESTAMP DEFAULT NOW(),
    completed_at      TIMESTAMP,
    results           JSONB                           -- detection results, findings
);
```

### 4. `honeytokens` — Decoy Center → Honeytokens
```sql
CREATE TABLE honeytokens (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_type        TEXT NOT NULL,                  -- aws_key, api_token, file, dns
    location          TEXT,                           -- where the token is planted
    triggered         BOOLEAN DEFAULT false,
    triggered_at      TIMESTAMP,
    source_ip         TEXT,                           -- IP that triggered the token
    created_at        TIMESTAMP DEFAULT NOW()
);
```

### 5. `automation_rules` — Automations → Rules
```sql
CREATE TABLE automation_rules (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              TEXT NOT NULL,
    description       TEXT,
    trigger_type      TEXT,                           -- e.g. "Honeypot Critical Alert"
    trigger_condition TEXT,                           -- e.g. "alert.severity == critical"
    priority          TEXT DEFAULT 'medium',          -- critical, high, medium, low
    enabled           BOOLEAN DEFAULT true,
    conditions        JSONB DEFAULT '[]',
    actions           JSONB DEFAULT '[]',
    last_triggered    TIMESTAMP,
    trigger_count     INTEGER DEFAULT 0,
    created_at        TIMESTAMP DEFAULT NOW(),
    updated_at        TIMESTAMP DEFAULT NOW()
);
```

### 6. `playbooks` — Automations → Playbooks
```sql
CREATE TABLE playbooks (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              TEXT NOT NULL,
    description       TEXT,
    steps             JSONB DEFAULT '[]',             -- ordered list of action steps
    enabled           BOOLEAN DEFAULT true,
    created_by        UUID REFERENCES users(id),
    created_at        TIMESTAMP DEFAULT NOW(),
    updated_at        TIMESTAMP DEFAULT NOW()
);
```

### 7. `playbook_runs` — Automations → Execution History
```sql
CREATE TABLE playbook_runs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    playbook_id       UUID REFERENCES playbooks(id),
    rule_id           UUID REFERENCES automation_rules(id),
    status            TEXT DEFAULT 'running',         -- running, completed, failed
    progress          INTEGER DEFAULT 0,              -- 0-100
    trigger_event     TEXT,
    started_at        TIMESTAMP DEFAULT NOW(),
    completed_at      TIMESTAMP,
    result            JSONB
);
```

### 8. `integrations` — Integrations page
```sql
CREATE TABLE integrations (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              TEXT NOT NULL,
    type              TEXT,                           -- SIEM Connector, EDR Connector, SOAR, TIP
    description       TEXT,
    status            TEXT DEFAULT 'active',          -- active, inactive, error
    api_endpoint      TEXT,
    auth_method       TEXT,                           -- api_key, oauth, basic
    config            JSONB DEFAULT '{}',
    auto_sync         BOOLEAN DEFAULT false,
    alert_on_fail     BOOLEAN DEFAULT false,
    last_sync         TIMESTAMP,
    error_message     TEXT,
    created_at        TIMESTAMP DEFAULT NOW(),
    updated_at        TIMESTAMP DEFAULT NOW()
);
```

---

## What we need from you

1. **Create the 8 tables above** (in the priority order if doing it in passes).
2. **Seed a few sample rows** each so the UI has something to render (2–5 rows
   is plenty for the Decoy/Policies/Automations/Integrations screens).
3. **Confirm column names match the DDL** — our queries reference these exact
   names. If anything differs from the DDL above, flag it so we update the
   queries.

Once tables 1–3 exist, the Response Policies write endpoints and the Decoy
Center Simulation/Honeytoken tabs work immediately (routes are already in
`server/routes/decoy.js` and `server/index.js`). Tables 5–8 need a small amount
of new route code on our side, which we'll add as soon as the tables land.

Thanks!
