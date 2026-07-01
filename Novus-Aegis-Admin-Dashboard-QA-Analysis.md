# Novus Aegis — Admin Dashboard: End-to-End Design QA Analysis

**Reviewer:** Lead QA (Design Lab)
**Artifact:** Figma flows for the Novus Aegis autonomous-SOC admin dashboard (10 modules + auth + settings)
**Purpose:** Surface every end-to-end flow, missing automation, transition, and state *before* we close the design phase, so dev and design can factor it into reasoning and planning.

---

## 1. Executive Summary

The design is strong on **static, happy-path, single-role screens** (Platform Admin, full access, populated with data). It is weak on the three things that actually make an autonomous-SOC product shippable and safe:

1. **State coverage** — empty / loading / error / success-confirmation states are absent almost everywhere. Every screen currently assumes "data exists and the request succeeded."
2. **Human-in-the-loop oversight of autonomy** — the product's core promise is autonomous action (auto-isolation, auto-mitigation, self-healing mutation), but there is no unified **approval queue**, no **global autonomy kill-switch**, and no **explainability/audit trail for AI decisions**. Approvals are *referenced* in four places but *designed* in none.
3. **Role-aware experience** — five roles are defined in RBAC, but every flow is drawn for Platform Admin. There is no design for what a Read-Only Auditor, SOC Analyst, Org Admin, or Incident Responder sees, can edit, or is blocked from.

These are not polish items — they are end-to-end flow gaps that will surface as rework in build if not closed now. Details and priorities below.

---

## 2. Scope & Implied Roles (call-outs)

**Implied user roles (from RBAC & Audit screen)** — the design must eventually specify per-screen behavior for each:

| Role | Implied scope | Design status |
|---|---|---|
| Platform Admin | Full platform control & config | ✅ All screens drawn for this role |
| Org Admin | Scoped to a single tenant | ❌ No scoped views designed |
| SOC Analyst | Threat monitoring, manual override, incident response | ❌ Not designed |
| Incident Responder | Isolation override, incident mgmt, evidence collection | ❌ Not designed |
| Read-Only Auditor | View-only, exports, audit logs | ❌ No read-only variants (toggles/edit/export must degrade) |

**Implied product concepts that are under-designed:**
- **Multi-tenancy** — "Organizations/Tenants" implies every data screen (Overview, Incidents, Alerts, Compliance) is either *global* or *tenant-scoped*. There is no tenant selector / context indicator, so it's ambiguous which one you're looking at.
- **Autonomy** — auto-isolation, auto-mitigation, self-healing mutation, autonomous actions. This is the product's spine and needs its own oversight surface (see §3.2).

---

## 3. Cross-Cutting Gaps (highest leverage — fix once, benefits every module)

### 3.1 Missing UI states (systemic)
Present on essentially no screen; needed on essentially every screen:
- **Empty / first-run states** — brand-new tenant with no alerts, no incidents, no cloud accounts connected, no templates, no orgs. Right now a fresh install renders as broken/blank charts.
- **Loading / skeleton states** — for every data table, chart, and drill-down.
- **Error states** — data fetch failure, permission denied, server error, with retry affordance.
- **No-results states** — after search or filter returns nothing (Alerts, Orgs, Templates, Audit).
- **Success / confirmation feedback** — toasts/snackbars after every save, toggle, export, resolve, or delete. Currently the user never gets told an action worked.

### 3.2 Autonomy oversight surface (missing product spine)
- **Unified Approval Queue / Inbox.** "Manual Approval Thresholds" (Orgs), "Require approval" (Policies), "Pending" templates (Honeypots), and "Notify Stakeholders" (Incidents) all imply pending items an admin must action — but there is **no single place** to see and clear them. Design a global approvals inbox (with the notification bell in §3.3).
- **Global autonomy controls / kill-switch.** No way to pause all autonomous actions platform-wide (or per tenant) during an incident. This is a mandatory safety control for an autonomous SOC.
- **Autonomous action review + explainability.** The Overview shows "Autonomous Actions (Last 24h)" as a chart, but you cannot click an action to see *what the AI did, why, on what evidence, and how to undo it*. Every autonomous action needs a detail + rollback path, mirroring the Incident rollback pattern.

### 3.3 Notifications (missing for an alert-centric product)
- No **global notification bell / center** anywhere, despite critical alerts, incidents, approvals, and spend/quota breaches. Design in-app + email, and specify integrations expected by SOC teams (**Slack, PagerDuty, Teams, webhook**).
- No **notification preferences** screen (the profile dropdown references "Preferences" but it's not designed).

### 3.4 Real-time / freshness
- Autonomous SOC data must be live. Add **"last updated" timestamps**, **auto-refresh / live indicators**, and a **connection-lost state** (WebSocket/polling). Currently everything looks like a static snapshot.

### 3.5 Permissions behavior
- Define, per screen, the **read / write / hidden** matrix for all five roles. Include **permission-denied UX**, **last-admin protection** (can't remove/downgrade the only admin), and **self-lockout prevention**.

### 3.6 Session, security & platform basics
- **2FA challenge at login is missing** — 2FA is enable-able in Settings, so there *must* be a verification-code step in the sign-in flow. Not designed.
- **SSO / SAML** — expected for an enterprise multi-tenant security product; absent.
- **Idle/session timeout + re-auth**, **active session management** (sign out other devices), **API keys/tokens** for integrations — all missing.

### 3.7 Responsiveness & accessibility
- All frames are desktop-only. Confirm whether **tablet/mobile** is in scope (on-call SOC responders often act from phones).
- Several statuses rely on **color alone** (red/amber/green). Ensure icon + text labels, dark-theme contrast, keyboard nav, and visible focus states.

---

## 4. Module-by-Module End-to-End Flow Analysis

For each: the flow as drawn → what's missing to make it end-to-end.

### 4.1 Authentication & Onboarding
**Drawn:** Join as Admin, Sign in, Reset Password, Create New Password.
**Missing to close the loop:**
- **Post-action confirmations:** "check your email" after Send Reset Link; success screen after Create Password; email verification after sign-up.
- **Error states:** invalid credentials, wrong/expired reset link, weak password, password mismatch, email already exists, rate-limit/lockout.
- **2FA verification step** during login (see §3.6).
- **Invite-acceptance flow** — RBAC has "Send invitation email immediately," so an invited user needs a distinct *accept-invite + set-password* path (different from open self-signup).
- **Governance question:** open "Join as Admin" self-registration on a tenant-isolated security product is a risk — should this be invite-only or domain-gated? Flag for product decision.
- Button loading states, ToS/privacy acceptance, live password-strength meter.

### 4.2 Overview / Dashboard (+ drill-downs)
**Drawn:** KPI cards, Alerts Distribution, Remediation, MITRE ATT&CK, latest Threat, High-Risk Entities, Autonomous Actions, MTTR/SLA/System Health, and drill-downs (Threat Intel, Alert Mgmt + Filter modal, Remediation, Entity Risk, MITRE, SLA, System Health).
**Missing:**
- **Empty/loading/error** for every widget (fresh tenant = empty charts).
- **Tenant context**: is this global or one tenant? Add selector/indicator (§2).
- **Actionability**: are KPI cards clickable? Can a critical alert be **acknowledged/assigned/actioned directly** from Overview? Alert Management has a Filter modal but **no bulk actions, no export-result confirmation, no saved views**.
- **Date range**: "Last 30 Days" — add custom range and per-widget vs global scope.
- **Autonomous Actions** widget → click-through to action detail + undo (§3.2).

### 4.3 Organizations / Tenants
**Drawn:** Tenant list; detail tabs (Overview, Features, Limits, Isolation Guardrails) with toggles, resource limits, approval thresholds.
**Missing:**
- **Create / edit / suspend / delete tenant** flows (a "Suspended" status exists in the list, but no action produces it). Tenant **offboarding + data-retention** handling.
- **Save semantics** on Features/Limits toggles — auto-save or explicit Save? Changing a tenant's limits/features needs **confirmation + audit** (impacts a customer).
- **Plan / billing change** flow; **invoices**; action on **"High" budget utilization** warning.
- **Approval landing**: "Manual Approval Thresholds" implies approvals — where are they actioned? (→ §3.2 queue).
- Empty state, search-empty state, pagination, per-tenant user management link.

### 4.4 Policies
**Drawn:** Policy stats, active policies with toggles, Policy Builder (IF/THEN, Dry Run, Impact Simulation, Save), operator/field dropdowns.
**Missing:**
- **Dry Run & Impact Simulation output screens** — the buttons exist but there's no result view (the whole point of testing a policy).
- **Edit / delete / duplicate** existing policy; **remove condition**; **validation** (empty values, conflicting/overlapping policies, precedence when multiple match).
- **Versioning & change history + rollback** — mandatory for controls that govern autonomous action.
- **Confirmation when disabling a critical policy** (e.g., turning off "High Threat Auto-Isolation" is a major safety change) → possibly its own approval.
- Save success/error, empty state, and **policy scope** (global vs per-tenant).

### 4.5 AI Governance (AI Control & Safety)
**Drawn:** Sliders (Realism, Human Simulation, Session Length), Safety toggles, Language Styles, Forbidden Topics, Add-Forbidden-Topic modal, metrics.
**Missing:**
- **No Save/Apply mechanism** for sliders/toggles — an AI-safety config *must not* silently auto-apply; needs explicit apply + confirmation + audit.
- **Add Forbidden Topic → Predefined Topics list UI** (radio is shown, the selectable list is not); duplicate/empty validation; remove-topic confirmation.
- **Hallucination Flags (23) is a dead metric** — it must drill into a **review queue** where flagged responses are inspected/actioned. Same for Failure Rate. This is the feedback loop that makes governance real.
- **Warning when disabling a safety boundary** (Block PII off = data-exposure risk).
- Reset-to-defaults, change history, and **scope** (global vs per-tenant).

### 4.6 Honeypot Templates
**Drawn:** Template grid (Approved/Pending), detail view (vuln modules, fake filesystem, banners, stats, human-sim, safe-mode).
**Missing:**
- **Approval workflow** — MySQL is "Pending" but there's no Approve/Reject action, approver, or reason capture (→ §3.2).
- **Create / edit / clone / delete / archive** template; **is the detail editable?** No Save.
- **Deploy flow** — "Usage Count 234" implies deployment to infrastructure, but there's no deploy action or target selection.
- **Versioning**, **filter/sort** (risk, usage, status), empty & no-results states, **preview/test before approval**.
- **QA data bug:** detail view lists **"CVE-2023-1234" twice** (identical modules) — verify data model / dedupe.

### 4.7 Infrastructure (Cloud & Infrastructure)
**Drawn:** Cloud accounts (AWS/Azure/GCP), regions, K8s clusters, spend limits, alert thresholds, quotas.
**Missing:**
- **Connect / disconnect cloud account** flow (IAM role / OAuth setup) — including empty first-run "no accounts connected" state.
- **Remediation action for warnings** — GCP shows IAM-Health warning; `prod-cluster-03` is at **92% / warning status** — no action (scale, alert, investigate) is designed.
- **Save + confirmation** for spend limits and alert thresholds; **auto-throttle toggle** impact confirmation; **what fires when a limit is hit** (notification, throttle event).
- **Quota-increase request** flow; cluster drill-down; historical cost trends.

### 4.8 RBAC & Audit
**Drawn:** Users list, Create-User modal, Roles & Permissions, Audit Trail (Load More).
**Missing:**
- **Edit user** destination (EDIT button target), **deactivate/suspend/delete**, **resend invite**, **pending-invite state** (invited ≠ active).
- **Custom roles?** Roles look fixed — clarify whether admins can create/edit roles & permissions.
- **Audit Trail filtering/search/export** (by user/date/action/target) and **audit detail** ("View Details" target).
- **Last-admin / self-lockout protection**, **MFA enforcement per role**, bulk actions, empty states.
- **QA layout bug:** the frame shows **two overlapping "Create New User" modals** — confirm this is an export artifact, not two states merged.

### 4.9 Incidents
**Drawn:** Incident list (Active/Mitigated/Resolved/MTTR), detail (automated mitigations, manual actions, rollback controls, quick actions), Mark-Resolved & Rollback modals. *(Good: empty-ish and resolved states partially considered here.)*
**Missing:**
- **Success feedback + state change** after Mark Resolved / Rollback; **rollback-failed error state**.
- **Notify Stakeholders** flow (recipients, channels, template) — button with no destination.
- **Export Incident Report** output/format; **Partial Rollback selection UI** (button exists, selector missing).
- **Assign/escalate**, **incident timeline & comments**, **post-incident review**, filter/search.
- **Taxonomy clarification:** Incidents here are *operational* (LLM outage, queue backlog) while Alerts are *security*. Make the distinction explicit in IA so users aren't confused about where a given event lives.

### 4.10 Compliance & Data Governance
**Drawn:** Retention settings, region/export controls, compliance reports (SOC2/ISO/GDPR-Under-Review/HIPAA), evidence exports, report modal (Share/Download).
**Missing:**
- **Export = async job**: progress, completion, download-ready notification, and **who exported (chain-of-custody audit)**. Currently a single button with no outcome.
- **Retention/region-lock changes** have legal weight → **confirmation + audit** (no save state shown).
- **Remediation tracking** — SOC 2 shows "4 controls need remediation" and GDPR is "Under Review," but there's no workflow to assign/track those.
- **Share Report** — recipients, link permissions, expiry.
- **DSAR / right-to-be-forgotten** flow (GDPR obligation for this data class); scheduled recurring exports; empty state.

### 4.11 AI Copilot
**Drawn:** Chat thread, input, Send.
**Missing:**
- **Typing/loading indicator**, **AI-unavailable error state**, **stop generation**, **regenerate**.
- **Message feedback (👍/👎)** — this is the human signal that should feed AI Governance's hallucination/failure metrics; **copy**, **citations/sources** for factual security claims (traceability), export conversation.
- **Conversation history / new chat / sessions**, suggested prompts, first-use empty state, token/rate limits.
- **Critical:** can the Copilot *take actions* or only advise? If it can act, every action needs **confirmation + audit** and must appear in the autonomy oversight surface (§3.2). Clarify and design accordingly.
- The **"Search organizations" bar** in the Copilot header seems out of place — confirm scope/purpose.

### 4.12 Settings
**Drawn:** Profile, Security (password, 2FA toggle).
**Missing:**
- **2FA enrollment wizard** — the toggle exists but there's no QR/secret, verify-code step, or **backup/recovery codes**; and **disable-2FA confirmation**.
- **Password change** validation/error/success; **email-change verification**; **avatar upload** flow.
- **Notification Preferences** and **Account Settings** (referenced in the profile dropdown) are undesigned; **session management**, **API keys**, **connected apps**, **delete account**.

---

## 5. Missing Automations (consolidated — the "does it act on its own?" list)

Factor these into reasoning and planning; each needs both the automation *and* its visible control + audit:

1. **Approval routing** — pending items (templates, policy actions, tenant thresholds) auto-route to an approver inbox with SLA. *(Currently no queue.)*
2. **Autonomous action logging + explainability + undo** — every AI/auto action recorded with rationale and one-click rollback. *(Currently a chart with no detail.)*
3. **Threshold-triggered alerts** — spend/quota/error-rate/capacity breaches auto-notify. *(Thresholds exist; the resulting notification/automation does not.)*
4. **Auto-throttle on spend limit** — toggle exists; confirm the event, notification, and reversal path.
5. **Hallucination/failure feedback loop** — flagged AI responses auto-populate a review queue; Copilot 👎 feeds it.
6. **Safe-mode fallback trigger** — honeypot "Safe Mode Fallback" is described; design the trigger event, notification, and admin visibility.
7. **Compliance/evidence export jobs** — async generation + ready notification + chain-of-custody audit.
8. **Retention auto-deletion** — "Data age > 90 days → auto-delete" policy exists; needs execution logging and a pre-deletion notice.
9. **Session/idle auto-logout** and **invite/reset link expiry** automations.

---

## 6. Visible Data / Consistency QA Flags (fix in mock + verify data model)

- Honeypot detail: **duplicate identical CVE-2023-1234** modules.
- Templates: **Apache and MySQL both show "Usage Count 421"** — likely copy-paste mock data; confirm.
- Compliance/audit dates are **2024** while system date is **2026** — validate date logic and relative-time rendering ("1 hour ago") against real timestamps.
- **Two overlapping Create-User modals** in the RBAC frame — confirm artifact vs intended.

---

## 7. Prioritized Backlog

**P0 — Blocks a safe, shippable end-to-end product (close before exiting design):**
- Autonomy oversight: approval queue, global kill-switch, autonomous-action detail + undo (§3.2)
- 2FA login challenge + invite-acceptance flow (§3.6, §4.1)
- Empty / loading / error / success-confirmation states for all data screens (§3.1)
- Role-based permission behavior across all screens (§3.5, §2)
- Save/confirm/audit semantics for governance, policy, tenant-limit, and retention changes (§4.4–4.5, 4.3, 4.10)
- Auth error states (§4.1)

**P1 — Needed for a complete, seamless daily workflow:**
- Global notification center + integrations (Slack/PagerDuty) (§3.3)
- Policy Dry-Run/Impact-Simulation output; policy versioning (§4.4)
- Hallucination/failure review queue + Copilot feedback loop (§4.5, §4.11)
- Tenant CRUD + suspend/offboard; tenant context selector (§4.3, §2)
- Honeypot approve/reject + create/deploy (§4.6)
- Cloud account connect + warning remediation (§4.7)
- Incident notify/escalate/assign + report export; RBAC user edit/deactivate + audit filter/export (§4.9, §4.8)
- Real-time/freshness indicators (§3.4)

**P2 — Completeness & maturity:**
- Notification preferences, session mgmt, API keys, 2FA enrollment wizard (§4.12)
- Compliance remediation tracking, DSAR, scheduled exports (§4.10)
- SSO/SAML (may rise to P1 depending on target customers) (§3.6)
- Responsive/mobile + accessibility pass (§3.7)
- Fix data/consistency bugs in mocks (§6)

---

## 8. Recommendations for Closing the Design Phase

1. **Add a state matrix to the design system.** For every component/screen: default, empty, loading, error, no-results, success — so state coverage stops being ad-hoc.
2. **Design the autonomy oversight surface as its own epic.** Approval inbox + kill-switch + action-explainability are the product's differentiator and its biggest current hole; they cut across five modules.
3. **Produce a role × screen permission matrix** and design at least the Read-Only Auditor and Org Admin variants before build — retrofitting permissions is expensive.
4. **Define save/confirm/audit conventions** (which changes auto-save, which need confirm, which need approval, all of which get audited) and apply uniformly — critical because these controls govern autonomous behavior.
5. **Clarify multi-tenancy scope and the Incidents-vs-Alerts taxonomy** in the IA so navigation and data scoping are unambiguous.
6. **Specify async job UX** (exports, report generation, deploys) once, reuse everywhere.
