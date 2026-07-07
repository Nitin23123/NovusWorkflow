# Web-app handoff → FastAPI `dev` (src/UI + src/backend)

Branch with everything: **`Nitin23123/NovusAegisAI` → `nitin/webapp-updates`**
(old `web-app/` layout: frontend under `web-app/src/`, backend under `web-app/server/`)

Mapping to the new `dev`:
- `web-app/src/**` → `src/UI/src/**` (frontend — I'll move this)
- `web-app/server/**` → `src/backend/**` (backend — Python port, mostly into `features/`)
- `migrations/*.sql` → root `migrations/` (run as-is)

---

## 1. Migrations (add to root `migrations/`, run as-is — plain SQL)
- **`700_agents.sql`** — `CREATE TABLE agents (id, name, type, enabled, instances, status, config jsonb, created_at, updated_at)` + seeds 6 agent types.
- **`701_indicator_flags.sql`** — `ALTER TABLE threat_indicators ADD COLUMN watchlisted bool, false_positive bool` (default false).

---

## 2. Backend endpoints to converge → `src/backend/features/*.py`

Most slot into modules that already exist (`settings.py`, `threats.py`, `audit.py`); **agents** + **notifications feed** are new.

### settings.py
- `GET /api/settings/profile` — scope to the JWT user (`req.user.sub`), **not** `WHERE role='admin'`; also return `avatar` (`avatar_url`).
- `PATCH /api/settings/profile` — accept `avatar`; write audit ("Updated profile", or "Enabled/Disabled 2FA" when `mfa` changes).
- `PATCH /api/settings/password` — **verify `currentPassword`** with bcrypt before updating (skip only if hash is null); write audit ("Changed password").
- `POST/PATCH/DELETE /api/settings/users` — **admin-gate** (`role === 'admin'` → else 403); write audit (Created/Updated/Deactivated user).

### threats.py
- `GET /api/threats/indicators` — return `watchlisted`, `false_positive`.
- `PATCH /api/threats/indicators/:id` — accept `watchlisted`, `false_positive` (booleans; mutually exclusive is enforced client-side).

### audit.py (helper + wiring)
- `writeAudit({userId, action, target, details, ip})` — inserts into `audit_logs` (best-effort, never throws). Capture client IP (honor `X-Forwarded-For`).
- Call it on: **login** (auth), **password change**, **2FA enable/disable**, **profile update**, **user create/edit/deactivate**, **agent configure/enable/disable**.
- `GET /api/settings/audit-logs` — add filters `?action=&userId=&from=&to=&limit=&offset=` + pagination.

### agents.py  (NEW module + `agents` table)
- `GET /api/agents`
- `PATCH /api/agents/:id` — instances / status / enabled / config — **admin-gate**, write audit.
- `POST /api/agents/:id/enable` · `POST /api/agents/:id/disable` — **admin-gate**, write audit.

### notifications (actions/feature)
- `GET /api/notifications` — **multi-source feed**: union of alerts (crit/high, unresolved), active attack_sessions, open investigations, active honeypots. Params `?type=alert|attack|investigation|honeypot|all&severity=&limit=&offset=`. Each item: `{id, type, title, desc, severity, time, read}`. Per-source try/except so a missing table degrades to empty.
- `GET /api/notifications/unread-count`, `POST /api/notifications/:id/read` (already exist).

### ai threads
- `PATCH /api/ai/threads/:id` — rename (`title`). (Delete already exists.)

### attacks.py
- `POST /api/attacks/:id/terminate` — `SET ended_at = COALESCE(ended_at, NOW())` (idempotent).
- `GET /api/attacks` — expose `honeypot_id` (for force-rotation).

### decoy
- `POST /decoy/fleet` (deploy honeypot), `POST /decoy/fleet/:id/rotate` (new IP/instance), `GET /decoy/fleet` (`WHERE status <> 'terminated'`), `POST /decoy/honeytokens`, `DELETE /decoy/honeytokens/:id`.
- `GET /decoy/simulations/:id` — resolve `target_honeypots` UUIDs → `{id, name, status}`.

---

## 3. Frontend → `src/UI/src/` (I'll move these; listing for reference)

**New files (drop-in, no conflicts):**
- `components/NotificationsView.jsx`
- `components/DeployHoneypotWizard.jsx`
- `components/DeployHoneytokenWizard.jsx`
- `components/HoneytokensTab.jsx`

**Modified (reconcile into your `src/UI` versions):**
- `App.jsx` — add `notifications` route
- `api/client.js` — new methods (`agents.*`, `notifications.list(params)`, `ai.renameThread`, `commandCenter`, `policies.*`), surface server error messages
- `components/Sidebar.jsx` — Notifications nav item (bell)
- `components/SettingsView.jsx` — 2FA setup (authenticator/email/SMS) + disable, session password verify, Agent Settings enable/disable/configure
- `components/ThreatIntelView.jsx` — relationship graph + node-details panel; feed-card Watchlist/False-Positive buttons
- `components/AskQuestionView.jsx` — session rename (persist) + delete (undoable)
- `components/DashboardView.jsx` — empty states, Threat Intel block workflow, `localhost:3001`→relative `/api`
- `components/CommandCenterView.jsx` — empty state, accent icons, `localhost`→`/api`
- `components/AttackSessionsView.jsx` — terminate / adjust isolation / force-rotate workflows
- `components/DecoyCenterView.jsx` — deploy wizards, honeytokens, simulation results
- `components/AlertsView.jsx` — `localhost`→`/api`
- `components/AutomationsView.jsx` — Execution History filter/search/pagination
- `components/InvestigationsView.jsx`, `InvestigationDetailView.jsx` — wired to API
- `components/ThreatActionsView.jsx` — wired to `/threats` (actors + IOCs)
- `components/ResponsePoliciesView.jsx` — raw `fetch` → `api.policies.*`
- `components/LoginPage.jsx` — forgot-password flow, 2FA verification (authenticator/backup/email/SMS)

---

## Notes for the port
- **Demo/simulated (UI-only, no backend yet):** 2FA setup codes (demo `123456`), threat-relationship graph nodes, threat-simulation result numbers, isolation-rules persistence, block-IP propagation. These don't need backend.
- **`mfa_enabled`** is a stored flag; login 2FA is a client gate (not enforced server-side) — real TOTP is a separate task.
