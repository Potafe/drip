# DRIP — AGENTS.md

> **DRIP** is a cloud security intelligence platform that combines automated Prowler scanning, AI-powered reasoning, and IaC correlation to surface, validate, and remediate AWS security findings.

---

## Project Overview

DRIP is built around one central concept: `scan_id`. Every log, finding, agent action, and repo correlation ties back to it. When in doubt, follow the `scan_id`.

The system is composed of two primary parts:

- **Backend** — Go (modular monolith, REST API + async workers)
- **Frontend** — Next.js (TypeScript, Tailwind, Framer Motion)

---

## Core Commands

### Backend (Go)

```bash
# From /src
make build          # Compile api + worker binaries
make run-api        # Start the HTTP server (cmd/api)
make run-worker     # Start the Asynq worker (cmd/worker)
make test           # Run all Go tests
make lint           # Run golangci-lint (config: .golangci.yml)
make fmt            # gofmt + goimports
```

> Always run `make lint` and `make test` before committing backend code.

### Frontend (Next.js)

```bash
# From /ui
yarn install        # Install dependencies
yarn dev            # Start dev server with HMR
yarn lint           # Run ESLint (config: eslint.config.mjs)
yarn build          # Production build — DO NOT run during agent sessions
yarn test           # Run test suite
```

> **Do not run `yarn build` inside an agent session.** It replaces the `.next` folder with production assets and breaks HMR.

---

## Repo Structure

```
.
├── src/                        # Go backend (monorepo root)
│   ├── cmd/
│   │   ├── api/main.go         # HTTP server entrypoint
│   │   └── worker/main.go      # Asynq worker entrypoint
│   ├── internal/
│   │   ├── config/             # Env + config loading
│   │   ├── domain/             # Core types: Scan, Finding, Resource, etc.
│   │   ├── store/              # Postgres + S3 interfaces and implementations
│   │   ├── api/                # Router, middleware, HTTP handlers
│   │   ├── scan/               # Prowler runner + output parser
│   │   ├── agent/              # Claude reasoning engine (NOT just an API wrapper)
│   │   ├── repo/               # Git clone + IaC indexer (Terraform, CDK, CF)
│   │   ├── graph/              # In-memory resource graph (nodes + edges)
│   │   └── queue/              # Asynq task definitions + handlers
│   └── pkg/prowler/            # Prowler JSON schema types (shared)
│
└── ui/                         # Next.js frontend
    ├── src/
    │   ├── components/
    │   │   ├── FindingsTable/  # Core findings view — severity, resource, confidence
    │   │   ├── InvestigationDrawer/  # Deep-dive panel with tabs + agent timeline
    │   │   ├── AttackSurfaceGraph/   # react-force-graph interactive visualization
    │   │   ├── CommandPalette/       # ⌘K jump to scan_id / resource / file
    │   │   └── Dashboard/            # Severity bars, trend sparklines, new vs resolved
    │   ├── pages/
    │   └── lib/
    └── public/
```

---

## Architecture

```
Frontend (Next.js)
        ↓
API Gateway (Go — REST)
        ↓
┌─────────────────────────────────┐
│ Scan Service   (Prowler runner) │
│ Agent Service  (Claude)         │
│ Repo Service   (Git clone)      │
│ Graph Service  (in-memory)      │
└─────────────────────────────────┘
        ↓
Queue (Redis / Asynq)
        ↓
Workers (Go goroutines)
        ↓
Storage (Postgres + S3)
```

**Infra stack:**
- Queue → Asynq (Redis)
- DB → Postgres
- Blob → S3
- Realtime → SSE (not WebSockets)
- Graph (v0.1) → in-memory; later → Neo4j

---

## The Agent Service (`internal/agent`)

> This is the most important service in DRIP. Treat it with extra care.

The agent is a **security analyst, not a chatbot**. It reasons over findings — it does not summarize them.

### Agent Input

```json
{
  "finding": {},
  "resource": {},
  "repo_context": {},
  "historical_scans": []
}
```

### Tools Available to the Agent

| Tool | Methods |
|------|---------|
| AWS Read Tool | `get_iam_policy(role_id)`, `get_s3_acl(bucket)` |
| Repo Tool | `search("s3 bucket config")`, `open_file("main.tf")` |
| Graph Tool | `get_connected_resources(resource_id)` |

### Agent Execution Steps

1. **Understand finding** — parse Prowler output
2. **Validate via AWS** — check real config via read-only AWS client
3. **Correlate with code** — find IaC origin (Terraform / CDK / CloudFormation)
4. **Assess risk** — exposure + blast radius
5. **Output structured result**

### Agent Output Format

```json
{
  "status": "CONFIRMED",
  "confidence": 0.92,
  "impact": "HIGH",
  "reasoning": [
    "IAM policy allows *",
    "Role attached to EC2 instance",
    "Instance has public IP"
  ],
  "timeline": [
    { "t": "10:21:01", "action": "Checked IAM" },
    { "t": "10:21:03", "action": "Parsed Terraform" },
    { "t": "10:21:05", "action": "Validated exposure" }
  ],
  "remediation": {
    "diff": "...",
    "explanation": "Restricts access to specific resources"
  }
}
```

### Prompting Conventions

**Don't do:**
```
"Analyze this finding."
```

**Do:**
```
"You are a cloud security auditor. Verify → cross-check → justify. Follow these steps: ..."
```

Force structure in prompts — reduces hallucination, improves confidence scores.

### Safety Constraints (non-negotiable)

- **No write APIs** — agent is read-only at all times
- **Max API calls per task** — enforce in queue config
- **Timeout per investigation** — set in Asynq task options

---

## Domain Vocabulary

These terms have specific meanings in DRIP. Use them consistently:

| Term | Meaning |
|------|---------|
| `scan_id` | Primary key tying all entities together (logs, findings, actions, repos) |
| `finding` | A raw Prowler output item |
| `verified finding` | A finding confirmed by the agent via AWS + IaC cross-check |
| `confidence` | Float 0–1 from agent output — how certain the agent is |
| `blast radius` | Resources reachable from a compromised resource (graph traversal) |
| `IaC origin` | The Terraform/CDK/CF file where a misconfigured resource is defined |
| `investigation` | A single agent run tied to one finding |

---

## Code Style

### Go

- Follow standard Go idioms — `gofmt`, `goimports`
- Errors: wrap with `fmt.Errorf("context: %w", err)` — never swallow
- Interfaces go in `domain/` — implementations in their respective service package
- No global state; inject dependencies via constructors
- Lint config is in `src/.golangci.yml` — do not bypass it

### TypeScript / Next.js

- TypeScript strict mode — no `any`
- Prefer `const` over `let`; never use `var`
- Use `interface` over `type` for object shapes
- Components: functional only, no class components
- Co-locate component styles with the component file
- Animations via **Framer Motion** only — do not add other animation libraries
- Graph: **react-force-graph** — do not swap it out

---

## Frontend UI Contracts

### Findings Table
- Columns: Severity, Resource, Status (`verified` / `unverified`), Confidence
- New findings: pulse animation
- Hover: scale + glow (Framer Motion)

### Investigation Drawer
- Tabs: `Overview` | `Agent Timeline` | `Code` | `Graph`
- Timeline: typewriter / streaming animation
- Confidence: rendered as a filled bar (e.g., `██████████ 92%`)
- Code: side-by-side diff view

### Command Palette (⌘K)
- Must be able to jump to: `scan_id`, `resource`, repo file path

### Attack Surface Graph
- Physics-based node movement
- Hover → highlight connected nodes
- Click node → open finding in Investigation Drawer

---

## Git Workflow

- Branch naming: `feat/<short-description>`, `fix/<short-description>`, `chore/<short-description>`
- Commits follow Conventional Commits (enforced via `commitlint.config.mjs` + Husky)
- PR title format: `[drip] <Title>`
- Always run `make lint && make test` (backend) and `yarn lint && yarn test` (frontend) before committing
- Pre-commit hooks defined in `ui/.husky/` — do not skip them

---

## Boundaries

- **Never call AWS write APIs** from within the agent or any service — read-only only
- **Never hardcode credentials** — all secrets come from env/config (`internal/config`)
- **Never bypass the queue** for long-running tasks — always dispatch via Asynq
- **Never use WebSockets** — realtime is SSE only
- **Never run `yarn build`** during an active agent session
- **Do not move** graph to Neo4j yet — in-memory only for v0.1

---

## Testing

- Backend: `make test` from `src/` — must be green before any merge
- Frontend: `yarn test` from `ui/` — must be green before any merge
- Add or update tests for any code you change, even if not asked
- Integration tests for the agent service go in `internal/agent/*_test.go`

---

## References

- Prowler JSON schema → `pkg/prowler/`
- Core domain types → `internal/domain/`
- Asynq task registry → `internal/queue/`
- See `CONTRIBUTING.md` for PR and review guidelines
- See `SECURITY.md` for vulnerability reporting