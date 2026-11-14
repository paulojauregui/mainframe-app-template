# AGENTS.md — AI & Agent Onboarding Guide

> **For Coding Agents and Engineers:** Read this document first before proposing or implementing changes. It defines the polyglot language strategy, architectural invariants, language-agnostic engineering standards, testing protocols, and PR preview delivery pipeline for services running on `mainframe`.

---

## 1. Project Mission & Mental Model

This repository is the **Golden Application Template** for new software and business ideas built autonomously by agents. It defines the contract between an autonomous agent's code and `mainframe`'s production Kubernetes environment (`192.168.13.100`):

1. **Inner Loop (In-Sandbox):** Agents implement features, add tests, and verify linting/types with zero external network dependencies.
2. **Review Loop (PR Previews):** Opening a Pull Request automatically triggers GitHub Actions to build `linux/amd64` images, while ArgoCD's `ApplicationSet` provisions an isolated preview namespace (`preview-<app>-pr-<number>`) and Gateway API `HTTPRoute` at `https://<app>-pr-<number>.paulojauregui.com`.
3. **Outer Loop (GitOps Promotion):** Human approval and merge to `main` promotes the service to persistent production and prunes the preview environment.

---

## 2. Polyglot Language Selection Strategy (Go-First Backends, Python for AI/ML, Rust for Systems)

When architecting a new service or evolving a repository from this template, select the implementation language from first principles:

| Tier / Language | When to Select | Key Runtime & Resource Characteristics |
| :--- | :--- | :--- |
| **1. Go (Default for Backend Services)** | **Default choice** for REST/gRPC APIs, microservices, relays, network/scraping workers, CLIs, and control planes that do not require Python-only ML training libraries. | Static `CGO_ENABLED=0` binary in `distroless`/`scratch` (`< 20 MB` image, `< 20 MiB` RAM, instant startup, native goroutines). Target newest stable Go toolchain (`1.23+`/`1.24+`). |
| **2. Python (AI/ML & Data-Science Workloads)** | Select when the service relies heavily on Python-native AI/ML/scientific libraries (PyTorch, vLLM, SDV/CTGAN, SciPy/NumPy, DuckDB/PyArrow analytical pipelines) or as an ML worker paired with a Go backend. | **Always use the newest Python version supported by dependencies** (prefer **Python 3.15**, or **Python 3.14** when upstream C-extension wheels require it; never default to legacy versions). |
| **3. Rust (Systems, Crypto & Zero-Cost Data Planes)** | Evaluate and select for ultra-low-latency engines, cryptographic/ZKP circuits, eBPF/kernel agents, and memory-critical data planes. | Static/minimal binary (`distroless-cc`), zero-GC deterministic tail latency, compile-time memory safety. |

---

## 3. Architectural Invariants (Language-Agnostic Engineering Standards)

Regardless of whether a service is written in **Go**, **Python**, or **Rust**, preserve these universal engineering invariants mapped to the target language's idiomatic toolchain:

### Rule 1: Strict Modularity & Standard Project Layout
* **Go**: Standard `cmd/<service>/main.go` entrypoint layout with private domain/application packages encapsulated under `internal/<package>/` (and `pkg/` only if explicitly exported). Use full module-path imports (`github.com/paulojauregui/<repo>/internal/...`).
* **Python**: Strict `src/<package_name>/` layout. Use **absolute imports exclusively** (e.g., `from mainframe_template.config import get_settings`). Relative imports (`from ..config`) are prohibited.
* **Rust**: Idiomatic crate/workspace layout (`src/main.rs`, `src/lib.rs`, or `crates/<name>/`) with explicit `use crate::...` imports and minimal `pub` surface area (`pub(crate)` by default).

### Rule 2: Deterministic Dependency Management
* **Go**: `go.mod` and `go.sum` maintained via `go mod tidy` and `go mod verify`.
* **Python**: `pyproject.toml` and `poetry.lock` managed via `poetry` (`poetry add` / `poetry add --group dev <pkg>`), targeting the newest supported Python (`>=3.14,<3.16` / `3.15`).
* **Rust**: `Cargo.toml` and `Cargo.lock` managed via `cargo`.

### Rule 3: Design by Contract (DbC) & Static Type Safety
* Every public function, method, struct/class, and interface across **Go, Python, and Rust** must include a structured doc comment (`// FuncName ...` in Go, `"""..."""` in Python, `/// ...` in Rust) that explicitly defines `Pre-conditions:` and `Post-conditions:`.
* Strict static typing is mandatory:
  * **Go**: Explicit types and small interfaces, zero unchecked `any` assertions, verified via `go vet ./...` and `golangci-lint` / `staticcheck`.
  * **Python**: Full type annotations on all parameters, return types, and attributes, verified via `mypy src tests` and `ruff check .`.
  * **Rust**: Strict type/lifetime safety, verified via `cargo clippy --all-targets -- -D warnings` and `cargo fmt --check`.
* **No Emojis**: Do not use emojis in source code comments, docstrings, ADRs, READMEs, or commit messages.

### Rule 4: Fast, Deterministic Unit Tests (Zero External Network Dependencies)
* Unit tests must run deterministically in sub-second time without requiring live network connections or external cloud endpoints (mock external HTTP/DB/LLM boundaries via interfaces):
  * **Go**: `go test -v -race -cover ./...` (table-driven tests with `testing` and `net/http/httptest`).
  * **Python**: `PYTHONPATH=src python3 -m unittest discover tests -p "*unittest.py"` and/or `poetry run pytest -v`.
  * **Rust**: `cargo test --all-targets`.

### Rule 5: Clean Container & Kubernetes Hygiene
* Containers must execute as a non-privileged user (`USER 10001:10001`). Never run as root.
* For **Go** services, compile a static binary (`CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w"`) in the builder stage and copy it into a minimal `gcr.io/distroless/static-debian12:nonroot` (or `alpine` with UID `10001:10001`) runtime stage.
* All HTTP services must expose `/healthz` (liveness) and `/ready` (readiness) probe endpoints on port `8000` (matching `deploy/base/01-deployment.yaml`).

### Rule 6: First-Class UI/UX Architecture (`@ux-architect`) & Product Design (`@product-designer`)
* User-facing services must integrate two dedicated design disciplines starting at **Gate 1 (before implementation)** rather than deferring frontend work to a single ad-hoc milestone:
  1. **Principal UI/UX & Interaction Architect (`@ux-architect` $\to$ `docs/UX_SPEC.md`):** Documents operator personas, core journeys, information hierarchy, keyboard-first navigation (`Tab` order, `:focus-visible`, `Escape` / `Cmd+K` bindings), WCAG 2.2 AA accessibility (semantic HTML5 landmarks, `aria-live`, `4.5:1` contrast), and the **5-State Component Matrix** (`Empty`, `Loading` with `CLS = 0` skeletons, `Nominal`, `Degraded/Offline Fallback`, and `Error` with single-click recovery).
  2. **Staff Product & Visual Systems Designer (`@product-designer` $\to$ `docs/DESIGN_SYSTEM.md`):** Documents cohesive design tokens (4-tier surface elevation hierarchy, semantic status hues, `4px`/`8px` spatial grid rhythm), typographic pairings (`--font-sans` + `--font-mono`), `font-variant-numeric: tabular-nums` on all numeric/financial/telemetry columns, and responsive breakpoints (`375px` mobile through `2560px` 4K).
* Decompose UI work into dedicated atomic frontend milestones (`1–2 UI/template/style files + 1 DOM/view test file` per `SPEC.md`) and complete a mandatory `@ux-architect` + `@product-designer` sign-off audit before opening any Pull Request.

---

## 4. Reference Directory Maps

### Option A: Go Service Layout (Preferred for Backends & APIs)
```text
<service-repo>/
├── AGENTS.md                    ← Invariants & agent instructions
├── README.md                    ← Service architecture & operations guide
├── go.mod / go.sum              ← Go module definition & cryptographic lockfile
├── Dockerfile                   ← Multi-stage CGO_ENABLED=0 distroless build (USER 10001:10001)
├── .github/workflows/ci.yml     ← go vet, go test -race, GHCR build, Trivy scan
├── cmd/
│   └── server/
│       └── main.go              ← HTTP/gRPC server entrypoint & graceful shutdown
├── internal/
│   ├── config/                  ← Typed environment configuration
│   ├── domain/                  ← Core domain models & DbC interfaces
│   └── api/                     ← HTTP handlers (/healthz, /ready, /api/v1/*) & tests
└── deploy/base/                 ← Kustomize manifests (01-deployment, 02-service, 03-httproute)
```

### Option B: Python Service Layout (For AI/ML & Data-Science Workloads)
```text
<service-repo>/
├── AGENTS.md                    ← Invariants & agent instructions
├── README.md                    ← Service architecture & operations guide
├── pyproject.toml               ← Poetry dependencies (Python 3.14/3.15), Ruff & MyPy config
├── Dockerfile                   ← Multi-stage non-root container build (USER 10001:10001)
├── .github/workflows/ci.yml     ← Ruff, MyPy, unittest/pytest, GHCR build, Trivy scan
├── src/<package_name>/
│   ├── __init__.py
│   ├── config.py                ← Typed Settings with environment overrides
│   └── main.py                  ← FastAPI / ASGI app with /healthz & /ready probes
├── tests/
│   └── test_api_unittest.py     ← Deterministic unit & integration tests
└── deploy/base/                 ← Kustomize manifests (01-deployment, 02-service, 03-httproute)
```

---

## 5. Local Verification Recipes

### Go Projects
```bash
gofmt -s -w .
go vet ./...
go test -v -race -cover ./...
kubectl kustomize deploy/base
```

### Python Projects
```bash
poetry check
poetry run ruff check .
poetry run mypy src tests
PYTHONPATH=src python3 -m unittest discover tests -p "*unittest.py"
poetry run pytest -v
kubectl kustomize deploy/base
```

### Rust Projects
```bash
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
kubectl kustomize deploy/base
```

---

## 6. PR Preview & Bot Authorship Protocol

* **Autonomous Identity:** All commits and PRs generated inside `agent-workshop` are authenticated via the `tron-agent-mainframe[bot]` GitHub App (`export GH_TOKEN=$(/home/tron/.local/bin/tron-github-token)`).
* **Branching Invariant:** Direct pushes to `main` are blocked by server-side branch protection. Always create a descriptive branch (`feat/...`, `fix/...`, `docs/...`).
* **CI Workflow Preservation:** Keep `.github/workflows/ci.yml` on `main` compatible with the repo language so feature branches do not need to modify `.github/workflows/*` unless explicitly permitted.
* **The Human Review Gate:**
  1. Once a PR is opened, automated CI checks validate formatting, static analysis, tests, and container builds, and ArgoCD deploys `https://<app>-pr-<number>.paulojauregui.com`.
  2. The human maintainer (@HowDiggy) reviews the diff and live preview, submits an **Approve** review, and merges to `main`.
