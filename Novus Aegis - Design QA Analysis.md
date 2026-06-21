# Novus Aegis — End-to-End Design QA Analysis

**Prepared by:** Design Lab QA
**Date:** 2026-06-21
**Reviewed:** 36 Figma-export PDFs in `PDF design/` + the existing build in `novus/` (React/Vite frontend + Express API)
**Purpose:** Identify every user workflow (entry point → final outcome), flag missing automations / transitions / interactive elements, evaluate whether the design supports a complete, seamless end-to-end experience, and give the design + dev teams a prioritized list of what to **factor into reasoning and planning before closing the design phase.**

---

## 0. How to read this document

- **P0 — Blocking.** Missing screen/state/flow that stops an end-to-end flow from working or forces engineering to invent UX. Must be resolved before design sign-off.
- **P1 — Significant gap.** The happy path works but a real user will hit a wall (edge case, error, missing affordance). Resolve before sign-off or schedule explicitly.
- **P2 — Hygiene / polish.** Consistency, placeholder, and copy issues that will leak into components if not cleaned up.

Findings are grouped into: **(A)** product-level cross-cutting gaps, **(B)** per-workflow findings, **(C)** a missing-flows checklist, **(D)** missing automations & transitions, **(E)** design↔build alignment, **(F)** design hygiene, **(G)** prioritized recommendations + a design-phase "definition of done."

---

## 1. Executive summary

The design set is **strong on the happy path and weak on everything that surrounds it.** Almost every frame is a single populated/default state. As a Lead QA, my headline conclusion is that the design **does not yet support a complete, seamless end-to-end experience** — three categories of work are missing:

1. **States.** Across all 36 files there are essentially no loading, empty, error, validation, or success/confirmation states. The product is real-time and action-heavy (autonomous SOC, live honeypot sessions, automated responses); without these states the experience will be inconsistent and the build will stall.
2. **Whole surfaces.** Several primary nav destinations have **no design at all**: **Alerts, Command Center, Automations, Ask a Question (AI assistant), and a Security/posture overview.** Alerts and Ask-a-Question are core to the SOC loop. The product's third advertised pillar — **Shadow AI Governance** — has no surface anywhere.
3. **Connective tissue.** Creation flows (deploy honeypot, create honeytoken, build a response policy, sign up), confirmation/result states for destructive actions, list management (search/filter/sort/pagination/bulk), cross-screen pivots (alert→investigation→session→attacker→IOC), RBAC/users-and-roles, and notifications are largely absent.

The autonomous "agent" model (Investigation Agents, Orchestrator, etc.) is a genuine differentiator and is partly designed, but the **human-in-the-loop control surfaces around the automations** (override, approval, audit of automated actions, failure handling) are thin.

**Implied user roles** (none are enforced in design or code today): **SOC Analyst** (primary persona — "Elevate"), **SOC Lead / CISO** (referenced as alert recipients in policies; needs escalation/oversight views), **Admin** (integrations, users, agents, audit, billing), and the **autonomous AI agents** themselves as non-human actors that take actions a human must be able to review and reverse. The lack of a roles model is itself a P0 gap (see A9).

---

## 2. Scope reviewed (surface map)

| Area | Design coverage | Notes |
|------|-----------------|-------|
| Auth / Onboarding | `Onboarding.pdf` (6 frames), `Update Password.pdf` | Auth only — **not** product onboarding; Sign-Up referenced but not drawn |
| Dashboard | `Dashboard.pdf` + `All Assets`, `Response Metrics` | Populated only; MITRE table all-zero |
| Attack Sessions | 5 tab files + `Modal-4/5/6` | Best-covered flow; live + actions present, states thin |
| Investigations | `Investigation.pdf` (~11 frames) + `Modal/-1/-2/-3` | List has filters/empty state; detail has 6 tabs; **two different** investigation detail designs (full page vs modal) |
| Threat Detection Details | `Threat Detection Details.pdf` + `-1` | Alert/threat detail modal with Execute actions |
| Threat Actions | `Threat Actions.pdf` | Read-only **threat-actor** library (not "actions") |
| Threat Intel | `Threat Intel.pdf`, `Frame 2085669092.pdf` | Feed + Graph (graph is a placeholder) |
| Recon Pro | `Recon Pro.pdf` (4 frames) | Scan flow present; results partly empty |
| Decoy Center | `Decoy Center.pdf` (11 frames), `-1`, `Component 6.pdf` | 4 tabs + detail; **no deploy / no honeytoken** |
| Response Policies | `Response Policies.pdf` | List present; **create/edit modal is a blank placeholder** |
| Integrations | `Integrations.pdf` | List + config + Configure modal (well covered) |
| Settings | 4 settings files + 2 mislabeled "Users & Roles" | No real users/roles; thin notifications |
| Components | `Progress Bar.pdf`, `Modal-7` (Configure Agent) | |
| **Alerts** | **NONE** | In nav + code, no design |
| **Command Center** | **NONE** | In nav + code, no design |
| **Automations** | **NONE** | In nav + code (Rules / Playbook Builder / Execution History), no design |
| **Ask a Question (AI assistant)** | **NONE** | The floating purple orb on every screen is its presumed launcher — never designed |
| **Shadow AI Governance** | **NONE** | Advertised product pillar, no surface at all |

---

## A. Product-level / cross-cutting gaps

### A1 — (P0) No loading / empty / error / validation / success states, product-wide
Every screen is a populated happy-path state. Concretely missing:
- **Loading / skeleton:** only Recon Pro shows one skeleton frame. Dashboard, all lists, all detail panels, all charts, all modals have no loading state.
- **Empty / first-run:** Investigations has a designed "No results" filtered state (good) and an empty Notes tab; everything else either shows fake data or, worse, a **blank undesigned panel** (Decoy → Honeypot Details → Attack Sessions is a bare dark card = a dead-end, not an empty state).
- **Error:** there is **no** "couldn't load," network-error, partial-failure, permission-denied, or rate-limited state anywhere — unacceptable for a product that depends on live integrations (Splunk/CrowdStrike/XSOAR/MISP) and real-time streams.
- **Form validation:** Login, Update Password, Create New Password, Adjust Isolation Rules, Configure Agent, Integration config, Create Policy — none show validation, mismatch, or field-error states.
- **Success / confirmation:** no toasts, no "Saved," no post-action confirmation anywhere.

**Why it matters:** This is the single largest risk to a "seamless experience." Engineering will invent these per-screen, producing inconsistency, and QA will have nothing to test against. **Factor in a full state matrix per screen as a design-phase deliverable.**

**Recommendation:** Adopt a standard 5-state pattern (Loading / Empty / Populated / Error / Success) and a standard inline-validation pattern, designed once as components and applied to every screen + modal.

### A2 — (P0) Destructive & high-impact actions lack confirmation and result states
| Action | Confirm designed? | Loading/Success/Error after? | Safeguard? |
|--------|-------------------|------------------------------|------------|
| Terminate Attack Session (`Modal-4`) | Yes | **No** | No typed/2nd-factor confirm for an "irreversible" act |
| Force Honeypot Rotation (`Modal-6`) | Yes (with note) | **No** (no migration progress/success) | — |
| Adjust Isolation Rules (`Modal-5`) | Form only | **No** | — |
| Delete Response Policy (trash icon) | **No** (blank modal) | **No** | — |
| Disable Agent (Agent Settings) | **No** | **No** | None for stopping an automation |
| Terminate Honeypot (Fleet) | **No** | **No** | — |
| Block IP / Isolate Host / Create Ticket (`Threat Detection Details`) | **No** | **No** | No undo for blocking a (possibly legit) IP |

**Why it matters:** These are containment actions in production security operations — blocking the wrong IP or terminating the wrong session has real impact. The user needs confirm → in-progress → result → (undo where possible). **Factor reversible vs irreversible action patterns into planning.**

### A3 — (P0) Whole nav destinations have no design
**Alerts, Command Center, Automations, Ask a Question, Security/posture** are all in the sidebar and partially built in code, but have **no design**. Most critical:
- **Alerts (P0):** the front door of SOC triage. The alert→investigate→respond loop cannot be validated without it. Code has an Alerts list + an Investigate modal, but no design governs it.
- **Ask a Question (P0):** the AI assistant is a headline capability and the floating orb appears on *every* screen, yet its panel, conversation, sources/citations, and "create investigation from answer" flow are undesigned (and mocked in code).
- **Automations (P1):** code has Automation Rules / Playbook Builder / Execution History. Undesigned, and its **relationship to Response Policies is undefined** (see A4).
- **Command Center (P1):** real-time ops view, undesigned.
- **Security / posture overview (P2):** code routes to a static placeholder.

### A4 — (P1) Overlapping concepts not reconciled: Response Policies vs Automations vs Agents vs Threat Actions
The product has at least four "automation-ish" surfaces that overlap and are not differentiated in the design:
- **Response Policies** = IF (conditions) → THEN (ordered actions).
- **Automations** = Rules + Playbook Builder + Execution History (code-only).
- **Agent Settings** = Investigation/Analysis/Orchestrator agents with instance counts.
- **Threat Actions** = *named* as actions but actually a read-only threat-actor library.

**Why it matters:** Users (and developers) will not know where to create an automated response, what's a "policy" vs a "playbook" vs an "agent," or where execution history lives. **Factor a single information-architecture decision** that defines these four and their boundaries before building any of them. Also rename "Threat Actions" if it is an actor library, or build the actual actions surface.

### A5 — (P1) The omnipresent floating AI orb is undefined
A purple circular FAB appears bottom-right on essentially every screen. It is almost certainly the "Ask a Question" launcher, but it has **no label, no tooltip, no hover/active state, no open/expanded panel, and no defined relationship to the Ask-a-Question nav item.** Define it as a first-class component: trigger, panel, context-awareness (does it know what screen I'm on?), and how it differs from the full page.

### A6 — (P1) Creation / deploy flows missing for primary resources
| Resource | "Create/Deploy" entry in design? | Evidence it should exist |
|----------|----------------------------------|--------------------------|
| Honeypot / Decoy | **No** CTA anywhere | Cards show "Deployed By: AI / Manual" → a manual deploy flow is implied |
| Honeytoken | **No** tab/list/flow at all | Honeytokens are in product scope + DB + status log |
| Response Policy | Button yes, **builder blank** | "+ Create Policy" opens an undesigned gray modal |
| Investigation | **No** "New Investigation" | All investigations appear AI-created; manual create may be needed |
| Sign Up / account | **No** screen | Linked from Login |
| Integration | **Yes** (Configure modal) | ✅ good reference pattern |

The Integration "Add/Configure" modal is the model to follow; the others need the same treatment.

### A7 — (P1) List/table management affordances missing across the board
Most lists lack search / filter / sort / pagination / bulk-action:
- Decoy lists (honeypots, attackers, fleet), Threat Actions, Threat Intel feed, Audit Logs ("Load More" only), All Assets — none have full management controls.
- **Investigations list is the exception** (has filters, search, date range, pagination) — use it as the standard. But it has **row checkboxes + select-all with no bulk-action bar** — selecting rows does nothing (P1 dead-end).

### A8 — (P1) Real-time behavior implied but its mechanics undesigned
Live Interaction shows "system is typing…", live timestamps, MONITORING status; Command Center has a live timer in code; dashboard counters read like live data. Yet there is no designed: connection-lost/reconnecting state, session-ended/attacker-disconnected transition, "no activity yet" live-empty, live vs paused indicator, or "last updated / refresh" affordance. **Factor the real-time transport (polling vs webssocket) and its UI states into planning** — they drive both design and architecture.

### A9 — (P0) No RBAC / users / roles, despite being clearly implied
- The two files named **"Users & Roles" actually render a single-user Profile page.** There is no user list, no role management, no permission matrix, no invite-user flow.
- Audit logs reference **multiple users** (admin@, sarah.chen@, mike.torres@) performing different action classes → teams + roles clearly exist.
- Investigations are **assigned** to people (Sarah Chen, Mike Torres) → ownership/assignment exists, but no assign UI.
- Destructive/containment actions imply privilege gating, but there's no role concept.
- Code confirms **no auth and no role-gating at all** — login is cosmetic.

**Why it matters:** This is an enterprise security product; who-can-do-what is a core requirement, and it intersects with A2 (who can terminate/block) and the "Shadow AI Governance" pillar. **Factor a real roles & permissions model (and the Users/Roles/Invite screens) into the design phase.**

### A10 — (P1) Notifications & in-app alerting are thin
- Notification Preferences = 5 event toggles + **Email only.** No SMS/Slack/Teams/PagerDuty/webhook/push/in-app; no per-event channel routing, severity thresholds, quiet hours, or escalation policy.
- There is **no in-app notification center / bell** in the app chrome — in a SOC, a critical incident must surface immediately regardless of which screen the analyst is on.

### A11 — (P1) Cross-screen pivots are implied but dead-end
The data model clearly wants a connected investigation graph, but the links aren't designed:
- Threat Detection Details → no link to a full Investigation.
- Attribution "Matched Threat Actor Profiles" / Threat Actions "Related Incidents" / Threat Intel IOCs / Top-Assets rows → look clickable, **no destinations.**
- Investigation "View Response →" → automated-response detail not designed.
- Alert → Attack Session pivot exists in code but the Alert detail is undesigned.

Define the **pivot map** (alert ↔ investigation ↔ attack session ↔ attacker profile ↔ threat actor ↔ IOC ↔ asset) so every entity links to its neighbors.

### A12 — (P1) "Onboarding" is auth only — no product onboarding / first-run
`Onboarding.pdf` is the **authentication** flow (Sign In, OTP, Reset, Check Email, New Password). Missing as a flow:
- **Sign Up / registration** (referenced, not drawn).
- **Org / workspace creation**, first-admin setup.
- **First-run "time to value":** connect first integration, deploy first honeypot, invite team — and the **empty first-run states** of the dashboards that follow.
- Auth edge cases: wrong-password, locked account, SSO failure, **expired/invalid reset link**, OTP invalid/expired + resend countdown, and the post-success landing screen (every submit button currently leads to an undrawn destination).

### A13 — (P2) Accessibility, responsive, and density are unaddressed
No keyboard/focus states, no responsive/breakpoint frames (these are very wide artboards), no dark/light handling shown, no guidance on dense-table behavior or text truncation/tooltips (several asset/email values are truncated with no expand). Worth a checklist pass before sign-off.

---

## B. Per-workflow findings (entry → outcome, with edge/empty/error gaps)

### B1 — Authentication (entry: Sign In → outcome: Dashboard)
**Designed:** Sign In, OTP Verification, Reset Password, Check Your Email, Create New Password.
**Gaps:** No Sign-Up; no post-login landing/success; no error/validation states; OTP entry context (signup vs login MFA) undefined, no expired/resend-timer state; Create-New-Password has no success or back-to-login; Check-Your-Email has no resend; no expired-reset-link handling; no "remember me"; no show/hide on login password. **(P0 for states + sign-up; P1 for the rest.)**

### B2 — Alert triage → investigation → response (the core SOC loop)
**Designed:** Investigations list (with filters + empty state) and a 6-tab detail; an Investigation **modal** (4 tabs); Threat Detection Details modal with Execute actions.
**Gaps:** **Alerts page itself is undesigned (P0).** Two competing investigation-detail designs (full-page 6-tab vs modal 4-tab) — pick one (P1). No manual **Create / Assign / Escalate / Resolve / Reopen** actions (conclusions appear AI-set only) — confirm whether humans ever drive these (P1). Notes tab has no "add note" input (P1 dead-end). Row checkboxes with no bulk action (P1). "View Response →" and "See Alert" dead-end (P1). No SLA/age, ownership, or hand-off UI.

### B3 — Live attack engagement (entry: Attack Sessions → outcome: contain/terminate)
**Designed (best-covered flow):** 5 tabs (Live Interaction, Behavior Analysis, Self-Healing & Mutation, Attribution, Isolation & Response) + 3 action modals.
**Gaps:** Live stream has no connection-lost/ended/empty/reconnecting states (P0). Action modals (terminate/rotate/isolate) have no in-progress/success/error (P0, see A2). No pivot from a session to its Investigation / Attacker Profile / Alert (P1). Attribution self-contradicts (Unknown 28% vs APT28 94%) (P2). No way to manually intervene in the live terminal even though it's the product's signature capability — confirm that's intentional (read-only by design).

### B4 — Decoy lifecycle (deploy → monitor → mutate/rotate → terminate)
**Designed:** Honeypot Activity, Attacker Profiling, Threat Simulation, Honeypot Fleet + detail sub-tabs; rotate & terminate buttons; simulation launch + config + logs.
**Gaps:** **No "Deploy New Honeypot/Decoy" flow (P0)** despite "Deployed By: Manual." **No Honeytoken surface at all (P0)** though it's in scope/DB. Honeypot Details → Attack Sessions sub-tab is a blank dead-end (P1). Simulation has Launch but **no Stop/Cancel/Abort** while running (P1). Terminate/rotate confirmations + result states missing (P0, A2). No search/filter on any list (P1). Component 6 shows progress "100%" while status says "Initializing" (P2).

### B5 — Automated response policy lifecycle (create → enable → monitor)
**Designed:** Policy list (conditions → ordered actions), enable toggle, execution stats.
**Gaps:** **The create/edit rule-builder is a blank gray placeholder (P0)** — the condition/action builder is the heart of the feature and is entirely undesigned. No test/dry-run/simulate-policy (P1). No delete confirmation (P0, A2). Relationship to Automations undefined (P1, A4). No versioning/audit of policy changes (P2). Backend POST/PATCH/DELETE not implemented (E-section).

### B6 — Threat intelligence consumption (feed → IOC → action)
**Designed:** AI summaries, Feed Stream, Intelligence Graph tab, actor cards.
**Gaps:** **Intelligence Graph is a placeholder** (icon + counts, no real graph) (P0 if graph is a promised feature). No per-IOC actions (block / add to watchlist / investigate / mark false-positive) (P1). No filter/severity/time-range/refresh on the feed (P1). No drill-down/detail for any IOC or summary (P1). No pivot to actor or investigation (P1, A11).

### B7 — Recon scan (enter domain → scan → results → export)
**Designed:** skeleton, scan-start (domain + Start Scan + Recent Scans), loading splash, results (WHOIS).
**Gaps:** Results top stat cards are empty placeholders (P1). No scan **scope/type/options** step — single-field one-click only (confirm intent) (P1). Failed-scan path: a "Failed" status row exists but no failure detail / retry / error reason (P1). Export has no format/destination and no handler in code (P1). **No authorization/ownership guardrail for scanning external domains** — a security & legal concern for a recon tool (P1). Placeholder/typo data (telsa.com, EXAMPLE.COM WHOIS) (P2).

### B8 — Settings / admin (profile, password, 2FA, notifications, agents, audit)
**Designed:** Profile & Security, Notifications, Audit Logs, Agent Settings; Update Password; Configure Agent modal.
**Gaps:** **No real Users/Roles/Permissions/Invite (P0, A9).** 2FA "Enable" toggle has **no setup/verification flow** (QR, codes, backup codes) (P0). Agent "Disable" has no confirm (P1, A2). Two overlapping password submit buttons (form-level + page-level) — clarify (P2). Notifications thin (P1, A10). Audit Logs: no search/date/severity filter or export; "Load More" only (P1). Password stored plaintext in code (security bug — E-section).

### B9 — AI assistant ("Ask a Question")
**Designed:** Nothing (only the undefined orb).
**Gaps:** Entire surface undesigned (P0): conversation UI, suggested prompts, **source/citation display, confidence, "turn answer into an investigation/action," history/threads, and error/limit states.** Mocked in code today.

### B10 — Onboarding / first-run
See A12 — effectively missing as a product flow (P0/P1).

---

## C. Missing-flows checklist (quick triage)

| # | Flow | Status | Priority |
|---|------|--------|----------|
| 1 | Alerts list + alert detail | **No design** | P0 |
| 2 | Ask-a-Question AI assistant (panel + orb behavior) | **No design** | P0 |
| 3 | Loading/empty/error/validation/success state matrix (all screens) | **Missing** | P0 |
| 4 | Confirm + in-progress + result + undo for destructive actions | **Missing** | P0 |
| 5 | Deploy New Honeypot / Decoy | **No design** | P0 |
| 6 | Honeytoken surface (list + create) | **No design** | P0 |
| 7 | Response Policy create/edit rule-builder | **Blank placeholder** | P0 |
| 8 | Users / Roles / Permissions / Invite (real RBAC) | **No design** | P0 |
| 9 | 2FA setup/verification flow | **Missing** | P0 |
| 10 | Sign Up / registration + post-login landing | **Missing** | P0 |
| 11 | Shadow AI Governance pillar | **No surface** | P0/P1* |
| 12 | Automations (Rules / Playbook Builder / Execution History) | **No design** | P1 |
| 13 | Command Center | **No design** | P1 |
| 14 | IA reconciliation: Policies vs Automations vs Agents vs Actions | **Undefined** | P1 |
| 15 | Cross-screen pivots (alert↔investigation↔session↔actor↔IOC↔asset) | **Dead-ends** | P1 |
| 16 | Real-time states (live/ended/reconnect/empty) | **Missing** | P1 |
| 17 | List management (search/filter/sort/pagination/bulk) | **Mostly missing** | P1 |
| 18 | In-app notification center + multi-channel notifications | **Missing** | P1 |
| 19 | Intelligence Graph (real, not placeholder) | **Placeholder** | P1 |
| 20 | IOC actions (block/watchlist/investigate/false-positive) | **Missing** | P1 |
| 21 | Manual investigation actions (create/assign/escalate/resolve/note) | **Missing** | P1 |
| 22 | Simulation stop/cancel; Recon scan-scope + failure/retry/export | **Missing** | P1 |
| 23 | First-run / onboarding empty states & setup | **Missing** | P1 |
| 24 | Accessibility / responsive / truncation-tooltip pass | **Missing** | P2 |

\*P0 if Shadow AI Governance is in the launch scope (it's in the product tagline); P1 if explicitly deferred.

---

## D. Missing automations & transitions (specifically)

The product is "autonomous," so the **automation feedback loop** must be visible and controllable. Currently missing:

1. **Automated-action transparency + reversal.** Investigations/sessions show automated containment ("Host isolated," "Credentials disabled," "Memory dump 67%") as read-only status. There is **no UI to pause, override, roll back, or approve** an automated action, and no failure state if an automation errors. (Ties to A2/A9.)
2. **Human-in-the-loop approval gates.** Policies can "Isolate Immediately" or "Alert CISO." Design needs an approval/escalation step for high-impact automations (who approves, what timeout, what default).
3. **Agent lifecycle automation.** Agent Settings shows instance counts + enable/disable, but no autoscaling/health/restart/error state for the agents themselves, and disabling an agent (stopping automation) has no warning about downstream impact.
4. **Scheduled/triggered automations surface.** Self-Healing says "Next mutation scheduled in ~8 minutes" and simulations have "Recent Runs," implying schedules/triggers, but there is no schedule editor or trigger configuration anywhere.
5. **Notification automation.** Critical events should auto-surface in-app (bell/center) and route by channel/severity — undesigned (A10).
6. **State transitions between screens.** Most transitions are implied, not designed: list→detail (entry/exit animations, back behavior, scroll restore), modal open/close, tab switching with unsaved changes, and the live→ended transition for sessions. Define the transition + unsaved-changes pattern once.
7. **Empty→populated transitions** for real-time panels (skeleton → first data → streaming).

---

## E. Design ↔ build alignment (so dev and design plan together)

The build is partly ahead of and partly behind the design — both teams should reconcile:

- **Response Policies:** UI calls `POST/PATCH/DELETE` but the backend implements **only GET** → create/toggle/delete **fail end-to-end**, *and* the create/edit modal is undesigned. Both layers incomplete for the same feature. (P0)
- **Ask a Question:** answer is **mocked client-side** (no LLM call, no POST endpoint) and undesigned. (P0)
- **Decoy actions:** the UI **fakes** rotate/run (real `POST .../rotate` and `POST .../simulations/:id/run` endpoints exist but aren't called); designs show the buttons but no result states. (P1)
- **Recon Pro:** backend endpoints exist but the UI **never calls them**; results stat cards are empty in the design. (P1)
- **Auth:** cosmetic in code (no credential check, no token), and the design also lacks auth states → **neither is real.** Passwords are stored **plaintext** server-side (security bug). (P0)
- **Blocked-on-DB features:** Automations, Threat Actions, Integrations write paths, Decoy Simulations + Honeytokens are blocked on missing DB tables (per `WORK_STATUS.md`) — these align with the design gaps above; sequence design + DB + API together.
- **Two investigation detail designs** (full page vs modal) both exist in design **and** code — converge on one.

---

## F. Design hygiene / consistency issues (P2 — clean before componentizing)

- Files **"Users & Roles.pdf"** and **"Users & Roles-1.pdf"** both render the **Profile** page (mislabeled).
- **Identity mismatch:** profile shows "Alex Dura" while the sidebar footer shows "Elevate" across the app.
- **Contradictions in sample data:** Attribution Unknown 28% vs APT28 94%; Component 6 progress 100% vs "Initializing"; Self-Healing timestamps (21:xx) vs session (12:34:xx); All Assets 52.90.115.168 = 2 alerts vs 1 on the dashboard card.
- **Typos:** "CrowedStrike," "Thirst Canary" (likely Tripwire/Canary), "Defence Evasion," "Controal," "View Documenation."
- **Placeholder leakage:** Recon shows "telsa.com" and EXAMPLE.COM/IANA WHOIS in a "tesla.com" result.
- **Truncated values with no tooltip/expand** (asset names, emails) across Dashboard/All Assets.
- **Dashboard MITRE Tactic table is all zeros** — decide if that's an empty-data state and design it as one.
- `Frame 2085669092.pdf` / `Progress Bar.pdf` are loose components (actor card, progress bar) with no states — fold into the design system with proper variants.

---

## G. Prioritized recommendations & planning recommendations

### Do first — P0 (blocks a working end-to-end experience; resolve before design sign-off)
1. **Design a state matrix** (Loading/Empty/Populated/Error/Success + inline validation) as reusable components, then apply to every screen and modal. *(A1)*
2. **Design the Alerts surface** (list + detail + triage actions) — without it the core SOC loop can't be validated. *(A3, B2)*
3. **Design the destructive-action pattern** (confirm → in-progress → success/error → undo where possible; reversible vs irreversible) and apply to terminate/rotate/isolate/disable/delete/block. *(A2)*
4. **Design the Response Policy rule-builder** (condition + action builder, validation, test/dry-run). *(B5)*
5. **Design Decoy creation** (Deploy Honeypot wizard) and the **Honeytoken surface** (list + create). *(A6, B4)*
6. **Define and design RBAC** (Users, Roles, Permissions, Invite) and decide which actions each role can take. *(A9)*
7. **Design the Ask-a-Question assistant** (panel, orb behavior, sources/citations, "create investigation/action," history, errors) and **wire the real LLM** path. *(A5, B9, E)*
8. **Complete the auth flow** (Sign Up, post-login landing, all error/validation states, 2FA setup, expired-link/OTP handling) and **build real auth** (no plaintext passwords). *(A12, B1, E)*
9. **Decide Shadow AI Governance scope** for launch and, if in scope, design its surface. *(A3)*

### Do next — P1 (significant gaps; schedule explicitly)
10. **Reconcile the automation IA** (Policies vs Automations vs Agents vs Threat Actions) and design Automations (Rules/Playbook/History) + Command Center. *(A4, A3)*
11. **Design real-time states** and decide the transport (polling vs websocket) with design + eng together. *(A8)*
12. **Standardize list management** (use Investigations as the reference) incl. the missing bulk-action bar. *(A7, B2)*
13. **Design the pivot map** so every entity links to its neighbors; remove dead-end clickables. *(A11)*
14. **Design notifications** (in-app center/bell + multi-channel + severity routing). *(A10)*
15. **Make the Intelligence Graph real** and add **IOC actions**. *(B6)*
16. **Add manual investigation controls** (create/assign/escalate/resolve/reopen/add-note) — confirm the human-vs-AI authority model. *(B2)*
17. **Complete Recon (**scope step, failure/retry, export format, scan authorization**) and Simulation stop/cancel.** *(B7, B4)*
18. **Converge the two investigation-detail designs** into one. *(B2, E)*

### Then — P2 (hygiene & polish)
19. Fix mislabeled files, identity mismatch, contradictory sample data, typos, placeholder leakage, truncation tooltips, and fold loose components into the design system with variants. *(F)*
20. Accessibility / responsive / focus-state / density pass. *(A13)*

### Definition of done for the design phase (acceptance checklist)
Before declaring design complete, every screen should have: ✅ all 5 states designed · ✅ every interactive element's target/result defined (no dead-end clickables) · ✅ confirm + result states for every action · ✅ create/edit/delete designed where the list implies them · ✅ role/permission behavior noted · ✅ real-time/refresh behavior noted · ✅ entry point and exit/return defined · ✅ empty/first-run state designed · ✅ realistic, internally consistent sample data.

---

### Appendix — implied user roles to validate with stakeholders
- **SOC Analyst** (primary; "Elevate") — triage, investigate, monitor sessions, run simulations.
- **SOC Lead / CISO** — referenced as alert/escalation recipients; needs oversight, approval of high-impact automations, and reporting.
- **Admin** — integrations, users/roles, agents, audit, billing/account.
- **Autonomous AI agents** (Investigation, Analysis, Threat-Intel, Orchestrator, etc.) — non-human actors that take real actions; humans must be able to **see, approve, override, and reverse** what they do. This is the crux of an "autonomous SOC" and is currently under-designed.
