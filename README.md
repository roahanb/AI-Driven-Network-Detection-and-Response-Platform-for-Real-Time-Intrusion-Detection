# AI-Driven Network Detection &amp; Response — Full SOC Platform

A production-ready, multi-tenant **NDR + XDR + SOAR** SaaS platform — built end-to-end to mirror commercial SOC products (Trend Vision One, CrowdStrike Falcon, Splunk SOAR).

It combines live Suricata packet capture, a hybrid ML ensemble (BiLSTM + XGBoost + Isolation Forest), MITRE ATT&amp;CK mapping, real-time WebSocket streaming, and a complete analyst workbench with **AI Companion, AI Agents (Triage / Investigate / Respond / Report), SOAR Playbooks, Case Management, and Threat Hunting**.

**Live at:** [roahacks.com](https://roahacks.com)

---

## What It Does

- **Real-time capture** — monitors your network interface via **Suricata EVE JSON** and silently filters background noise (mDNS, ARP, local UDP keep-alives)
- **Auto-classifies every event** with a 4-model ML ensemble producing a 0–1 threat score, attack category (12 classes) and risk tier
- **MITRE ATT&amp;CK mapping** for every incident (14 tactics, 17+ techniques)
- **Auto-triage** — Low traffic logged silently, Medium goes to dashboard monitoring, High raises a live alert via WebSocket
- **AI Companion** — analyst chat assistant with full incident context (Anthropic Claude, with rule-based fallback)
- **AI Agents** — one-click Triage, Investigation, Response, and Executive Report generators per incident
- **SOAR Playbooks** — visual condition/action builder, auto-runs on every new incident, full audit trail
- **Case Management** — investigation cases with chain-of-custody evidence, timeline, severity, assignees
- **Threat Hunting** — visual query builder + Top-N aggregation + saved hunts over historical incidents
- **Multi-tenant SaaS** — strict org isolation, RBAC (ADMIN / ANALYST / VIEWER), JWT auth

---

## Feature Map

| Module | Backend | Frontend |
|---|---|---|
| Live capture &amp; ML detection | `backend/main.py`, `inference.py`, `mitre.py`, `suricata/ndr_watcher.py` | dashboard incidents tab |
| **AI Companion** (chat assistant) | `backend/ai_companion.py` | `frontend/src/AICompanion.js` (floating launcher) |
| **XDR Workbench** (KPIs, asset risk, correlation clusters) | `backend/xdr_workbench.py` | `frontend/src/XDRWorkbench.js` |
| **AI Agents** (triage, investigate, respond, executive report) | `backend/ai_agents.py` | `frontend/src/AIAgents.js` |
| **SOAR Playbooks** (rule engine + auto-run on ingest) | `backend/soar.py` | `frontend/src/SOARPlaybooks.js` |
| **Case Management** (timeline, evidence, assignees) | `backend/cases.py` | `frontend/src/CaseManagement.js` |
| **Threat Hunting** (query builder, aggregation, saved hunts) | `backend/threat_hunt.py` | `frontend/src/ThreatHunting.js` |
| Auth + RBAC + multi-tenancy | `backend/auth.py`, `security.py`, `models.py` | login/register screens |
| Real-time push | `backend/websocket_manager.py` | live status indicator |

All AI features (Companion + Agents) gracefully fall back to deterministic rule-based output when `ANTHROPIC_API_KEY` is not set, so the platform is fully functional offline.

---

## Accuracy Results (Table II — Research Paper Alignment)

| Model | Accuracy | Paper Target |
|---|---|---|
| Isolation Forest | 90.6% | 91.5% |
| One-Class SVM | 64.8% | 89.7% |
| XGBoost Direct | 99.9% | ~95% |
| BiLSTM (2×128) | 22.7%* | 94.8% |
| **Ensemble (final)** | **99.8%** | **96.4%** |
| **Ensemble Macro-F1** | **99.7%** | **95.9%** |

*BiLSTM accuracy improves significantly with larger sequence datasets on GPU. The ensemble meta-learner compensates via XGBoost weighting.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend API | FastAPI (Python 3.11) |
| Live Capture | Suricata 8.x — EVE JSON stream on `en0` |
| ML — Sequence Model | BiLSTM (2×128 units, dropout 0.3, seq_len=20) — TensorFlow/Keras |
| ML — Anomaly Detection | Isolation Forest + One-Class SVM — scikit-learn |
| ML — Ensemble | XGBoost meta-classifier (fuses all model outputs) |
| Feature Engineering | 78-dimensional CICIDS2018 feature vectors, robust z-score normalization |
| Training Data | Synthetic CICIDS2018 + UNSW-NB15 (60,000 samples, 12 attack classes) |
| LLM Integration | Anthropic Claude (`claude-haiku-4-5-20251001`) — graceful rule-based fallback |
| Authentication | JWT (HS256) + bcrypt |
| Database | SQLite (dev) / PostgreSQL 16 (prod) via SQLAlchemy ORM |
| Real-time | WebSocket (`uvicorn[standard]`) per-org broadcast |
| Frontend | React 18, Axios |
| Container | Docker + Docker Compose |
| Threat Framework | MITRE ATT&amp;CK (14 tactics, 17+ techniques) |
| Log Formats | Suricata EVE JSON (live), Zeek TSV, CSV, plain text |

---

## ML Architecture

### Ensemble Stack (Section 4.2 — Research Paper)

```
Network Interface (en0)
     │
     ▼
Suricata (EVE JSON stream)
     │
     ▼
NDR Watcher (ndr_watcher.py) — noise filter + event batching
     │
     ▼
Feature Engineering (78 features — CICIDS2018 standard)
     │
     ├── Isolation Forest ──────────────────────┐
     ├── One-Class SVM ─────────────────────────┤
     ├── XGBoost (direct 12-class classifier) ──┤──▶ XGBoost Meta-Ensemble ──▶ Final Prediction
     └── BiLSTM (2×128, seq_len=20) ────────────┘        (threat score 0–1, attack category, risk tier)
                                                                │
                                                                ▼
                                                  MITRE ATT&CK mapping → Incident
                                                                │
                                                                ▼
                                                     SOAR Playbook auto-eval
                                                                │
                                                                ▼
                                                  WebSocket broadcast to org
```

### 12 Attack Categories (CICIDS2018)
`Benign` · `DoS-Hulk` · `PortScan` · `DDoS` · `DoS-GoldenEye` · `FTP-Patator` · `SSH-Patator` · `Bot` · `Web-BruteForce` · `Infiltration` · `Heartbleed` · `Ransomware`

### Automated Risk Tiers

| Tier | Risk Level | Threat Score | Auto Action |
|---|---|---|---|
| Tier 1 | Low | < 0.4 | Silently logged (background) |
| Tier 2 | Medium | 0.4 – 0.75 | Shown on dashboard (Monitoring) |
| Tier 3 | High | > 0.75 | Highlighted alert + WebSocket push (Active) |

No manual approval required for triage — all classification is automated. Analyst approval is reserved for response actions through SOAR / case workflows.

---

## SOC Workbench (Trend Vision One-style)

The dashboard exposes **8 tabs**:

| Tab | What you do there |
|---|---|
| **XDR Workbench** | KPIs, top risky assets, correlation clusters (campaigns of related incidents), MITRE coverage, 7-day trend, drill-down drawer |
| **Incidents** | Raw incident table with search, filter, MITRE columns, approve/reject |
| **Cases** | Open investigation cases, link incidents, attach evidence (IOCs, PCAP refs, notes), comment timeline, assign owners |
| **Threat Hunting** | Visual query builder over all historical incidents — filter by IP / domain / MITRE / AI score / time window, run Top-N aggregations, save hunts for re-runs |
| **SOAR Playbooks** | Build no-code playbooks with conditions (risk_level, MITRE tactic, AI score…) + actions (tag, escalate, notify, auto-approve, block_ip…). Auto-runs on every new incident; full execution history |
| **AI Report** | One-click executive summary (metrics, top threats, recommended priorities) — Claude-powered |
| **MITRE Heatmap** | 14-tactic heatmap, color-coded by incident count |
| **Analytics** | Risk timeline (last 10 days) |

A floating **AI Companion** is available on every screen for natural-language Q&amp;A about the org's posture, top threats, MITRE coverage, and response guidance.

---

## AI Agents

Each incident exposes four 1-click agents (visible inside the XDR Workbench drawer):

| Agent | Output |
|---|---|
| **Triage** | Priority (P1–P4), severity tier, attack stage, recommended tags |
| **Investigation** | Hypothesis, findings, IOCs, related incidents, suggested next steps |
| **Response** | Categorized recommended actions, urgency, automatable flag |
| **Executive Report** | Org-wide posture summary, top threats, KPIs, priorities for leadership |

All runs are persisted to the `agent_runs` audit table.

---

## SOAR Engine

- Conditions evaluated with **AND semantics**: `risk_level`, `mitre_tactic`, `alert_type_contains`, `source_ip_contains`, `destination_ip_contains`, `ai_score_min`, `threat_score_min`, `risk_tier_max`, `status`
- 8 built-in action handlers: `tag`, `escalate`, `notify`, `auto_approve`, `auto_reject`, `set_status`, `block_ip` (stub), `add_comment`
- Auto-execution wired into the `/upload-logs` ingest pipeline — every new incident is evaluated against every enabled playbook
- Manual `POST /api/v1/soar/playbooks/{id}/run` for ad-hoc runs
- Full execution audit log with matched conditions, action results, and timestamps

---

## Case Management

- Statuses: `Open` → `In Progress` → `Pending Review` → `Closed`
- Severities: `Critical` / `High` / `Medium` / `Low`
- Auto-logged timeline events: `created`, `comment`, `status_changed`, `assigned`, `incident_linked/unlinked`, `evidence_added/removed`, `closed`
- **Chain-of-custody evidence**: `ioc`, `log`, `artifact`, `note`, `pcap`, `screenshot` — JSON-shaped values
- VIEWER role read-only; only ADMIN can delete cases

---

## Threat Hunting

A free-form query engine over all historical incidents (org-scoped):

- Filters: source/dest IP (equals or contains), domain, alert_type, risk_levels[], statuses[], MITRE tactic/technique IDs[], attack_categories[], AI score range, threat score range, risk tier, time window (`last_n_minutes` or `start_time`/`end_time`), full-text search across `summary` + `ai_reason`
- Top-N **aggregations** by any of: source_ip, destination_ip, domain, alert_type, risk_level, status, mitre_tactic, mitre_technique, attack_category, risk_tier, ai_prediction
- **Saved hunts** — analyst-named, with run count and `last_run_at`; one-click replay
- `GET /api/v1/hunt/schema` exposes the filter/aggregate metadata for the UI builder

---

## Multi-Tenant SaaS Architecture

- Organization-isolated data — every query filtered by `organization_id`
- Role-based access control: `ADMIN` → `ANALYST` → `VIEWER`
  - VIEWER: read-only (no comments, no playbook edits, no case mutations)
  - ANALYST: full ops except destructive deletes
  - ADMIN: everything (including delete cases, delete incidents, control watcher)
- JWT access tokens with long-lived watcher service tokens (24h)
- First user per organization auto-assigned ADMIN

---

## Real-Time Operations

- WebSocket endpoint (`/ws?token=<JWT>`) — requires `uvicorn[standard]`
- Per-organization broadcast isolation
- Frontend auto-reconnects on disconnect
- Incidents appear on dashboard without page refresh
- Start/Stop watcher directly from the dashboard UI

---

## System Architecture

```
Browser
  │
  └── React SPA (localhost:3000 dev / Nginx prod)
        │
        ├── 8 dashboard tabs (Workbench, Incidents, Cases,
        │   Threat Hunting, SOAR, AI Report, MITRE, Analytics)
        ├── Floating AI Companion
        │
        └── FastAPI backend (localhost:8000)
              ├── Auth          /api/v1/auth/{register,login}
              ├── Incidents     /incidents
              ├── Watcher       /watcher/{start,stop,status}
              ├── WebSocket     /ws?token=<JWT>
              ├── Health/Metrics/health, /metrics
              │
              ├── AI Companion  /api/v1/ai/{chat,summary}
              ├── XDR Workbench /api/v1/xdr/{workbench,incidents/{id}}
              ├── AI Agents     /api/v1/agents/{triage,investigate,respond,report,history}
              ├── SOAR          /api/v1/soar/{playbooks,executions,action-catalog}
              ├── Cases         /api/v1/cases/...
              └── Threat Hunt   /api/v1/hunt/{query,aggregate,saved,schema}

Suricata (en0)
  └── eve.json → NDR Watcher → POST /upload-logs → ML ensemble → MITRE map
                                                               → SOAR auto-run
                                                               → Incident stored
                                                               → WS broadcast
```

---

## Database Schema (key tables)

| Table | Purpose |
|---|---|
| `organizations` | Multi-tenant containers |
| `users` | Auth + role mapping |
| `incidents` | Detected events with MITRE + ML output (org-isolated) |
| `playbooks` | SOAR rules — `trigger_conditions` (JSON) + `actions` (JSON list) |
| `playbook_executions` | SOAR audit log — matched conditions, action results |
| `agent_runs` | Every Triage/Investigate/Respond/Report invocation, with input + output |
| `cases` | Investigation cases (status, severity, assignee, tags JSON) |
| `case_incidents` | M:N case ↔ incident link |
| `case_events` | Auto-logged timeline (comments, status changes, assignments…) |
| `case_evidence` | Chain-of-custody attachments (type + label + JSON value) |
| `saved_hunts` | Analyst-saved threat hunts (query JSON, run_count, last_run_at) |

All new tables are auto-created via `Base.metadata.create_all(bind=engine)` on backend boot — no separate migration step.

---

## Running Locally

**Prerequisites:** Python 3.11+, Node 18+, Suricata 8+

```bash
git clone https://github.com/roahanb/AI-Driven-Network-Detection-and-Response-SaaS-Platform-with-Approval-Based-Response-Engine.git
cd AI-Driven-Network-Detection-and-Response-SaaS-Platform-with-Approval-Based-Response-Engine

# Backend
cd backend
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
pip install 'uvicorn[standard]' websockets
# Optional — enables Claude-powered AI Companion + AI Agents
export ANTHROPIC_API_KEY=sk-ant-...
uvicorn main:app --host 0.0.0.0 --port 8000 --reload

# Frontend (new terminal)
cd frontend && npm install && npm start

# Suricata (macOS)
brew install suricata
sudo suricata -c /opt/homebrew/etc/suricata/suricata.yaml -i en0 -D
```

Open **http://localhost:3000** → register → click **▶ Start** to begin live monitoring.
The very first user of an org is auto-promoted to ADMIN.

> **AI features without an API key**: Both AI Companion and AI Agents fall back to deterministic rule-based output, so every screen still works end-to-end with no key configured.

---

## API Endpoints

### Core
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/auth/register` | ✗ | Register user + org |
| POST | `/api/v1/auth/login` | ✗ | Login, returns JWT |
| POST | `/upload-logs` | ✓ | Ingest events (auto-runs SOAR after commit) |
| GET | `/incidents` | ✓ | List org incidents |
| PUT | `/incidents/{id}/approve` | ✓ ANALYST | Approve incident |
| PUT | `/incidents/{id}/reject` | ✓ ANALYST | Reject incident |
| DELETE | `/incidents` | ✓ ADMIN | Delete all incidents |
| POST | `/watcher/start` | ✓ ADMIN | Start NDR watcher subprocess |
| POST | `/watcher/stop` | ✓ ADMIN | Stop NDR watcher subprocess |
| GET | `/watcher/status` | ✓ | Watcher running state + uptime |
| WS | `/ws?token=<JWT>` | ✓ | Real-time incident stream |
| GET | `/health`, `/metrics` | mixed | Health + counters |

### AI Companion
| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/ai/chat` | Conversational analyst assistant with org context |
| GET | `/api/v1/ai/summary` | Executive posture summary |

### XDR Workbench
| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/xdr/workbench` | KPIs, top assets, correlation clusters, MITRE coverage, 7d trend |
| GET | `/api/v1/xdr/incidents/{id}` | Drill-down with evidence + related incidents + checklist |

### AI Agents
| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/agents/triage` | Priority/severity/stage/tags |
| POST | `/api/v1/agents/investigate` | Hypothesis + findings + IOCs + next steps |
| POST | `/api/v1/agents/respond` | Recommended response actions |
| POST | `/api/v1/agents/report` | Executive summary across the org |
| GET | `/api/v1/agents/history` | Audit log of past runs |

### SOAR
| Method | Path | Description |
|---|---|---|
| GET / POST | `/api/v1/soar/playbooks` | List / create playbooks |
| GET / PUT / DELETE | `/api/v1/soar/playbooks/{id}` | Single playbook ops |
| POST | `/api/v1/soar/playbooks/{id}/run` | Manual trigger |
| GET | `/api/v1/soar/executions` | Execution audit log |
| GET | `/api/v1/soar/action-catalog` | Builder metadata for the UI |

### Cases
| Method | Path | Description |
|---|---|---|
| GET / POST | `/api/v1/cases` | List (with `?status=`, `?assignee=`) / create |
| GET / PUT / DELETE | `/api/v1/cases/{id}` | Single case ops (DELETE = ADMIN) |
| POST | `/api/v1/cases/{id}/comments` | Add timeline comment |
| POST / DELETE | `/api/v1/cases/{id}/incidents/{incident_id}` | Link / unlink incident |
| POST | `/api/v1/cases/{id}/evidence` | Add evidence (ioc, log, pcap, …) |
| DELETE | `/api/v1/cases/{id}/evidence/{evidence_id}` | Remove evidence |
| GET | `/api/v1/cases/stats/summary` | Aggregate counts (open/closed/by severity) |

### Threat Hunting
| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/hunt/query` | Ad-hoc filtered hunt over incidents |
| POST | `/api/v1/hunt/aggregate` | Top-N group-by aggregation |
| GET | `/api/v1/hunt/schema` | Filter / aggregate / sort metadata |
| GET | `/api/v1/hunt/field-values/{field}` | Distinct values for a field (UI dropdowns) |
| GET / POST | `/api/v1/hunt/saved` | List / create saved hunts |
| GET / PUT / DELETE | `/api/v1/hunt/saved/{id}` | Single saved-hunt ops |
| POST | `/api/v1/hunt/saved/{id}/run` | Replay a saved hunt |

---

## Repository Layout

```
ai-ndr-platform/
├── backend/
│   ├── main.py               # FastAPI app, ingestion, watcher control
│   ├── ai_companion.py       # /api/v1/ai/* — chat assistant
│   ├── xdr_workbench.py      # /api/v1/xdr/* — KPIs, asset risk, correlation
│   ├── ai_agents.py          # /api/v1/agents/* — triage/investigate/respond/report
│   ├── soar.py               # /api/v1/soar/* — playbook engine + auto-run helper
│   ├── cases.py              # /api/v1/cases/* — case management
│   ├── threat_hunt.py        # /api/v1/hunt/* — query engine + saved hunts
│   ├── inference.py          # ML ensemble (BiLSTM + IF + OCSVM + XGB)
│   ├── mitre.py              # MITRE ATT&CK mapping
│   ├── auth.py / security.py # JWT + bcrypt + RBAC
│   ├── models.py / schemas.py / database.py
│   ├── websocket_manager.py  # per-org WS broadcast
│   └── requirements.txt
├── frontend/src/
│   ├── App.js                # 8-tab dashboard shell + routing
│   ├── AICompanion.js        # floating chat widget
│   ├── XDRWorkbench.js       # KPIs / clusters / asset risk + drawer
│   ├── AIAgents.js           # IncidentAgents + standalone AgentReport
│   ├── SOARPlaybooks.js      # playbook editor + executions tab
│   ├── CaseManagement.js     # case list + detail drawer (timeline/incidents/evidence)
│   └── ThreatHunting.js      # query builder + aggregation + saved hunts
├── suricata/ndr_watcher.py   # EVE JSON → backend ingest
├── docker-compose.yml / docker-compose.prod.yml
└── README.md
```

---

## Author

**Roahan B.**
[roahacks.com](https://roahacks.com)
