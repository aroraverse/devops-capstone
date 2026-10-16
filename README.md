# HelpDesk — DevOps Capstone

**Name:** Sarthak Arora  
**Roll No:** 24BCS10150  
**Repository:** https://github.com/aroraverse/devops-capstone

**HelpDesk** is a customer-support ticketing app. Support agents log tickets from customers, set a
priority (Low → Urgent) and category (Bug, Billing, Access, Other), move them through
`Open → In progress → Resolved → Closed`, and watch live KPIs (open, in-progress, resolved, urgent).

The point of the project is the delivery path: the same app runs on a laptop with Docker, is tested and
security-scanned in GitHub Actions, published to GHCR, deployed to Kubernetes (kind) with Helm (Ingress, HPA,
Prometheus/Grafana) and kept in sync by Argo CD (GitOps). Terraform describes the AWS VPC + EKS target.

```text
Developer ── git push ──► GitHub Actions
                            ├─ pytest + coverage, React build
                            ├─ SAST  (CodeQL)
                            ├─ SCA   (pip-audit, npm audit)
                            ├─ Secret scan (Gitleaks)
                            ├─ IaC scan (Trivy config) + terraform fmt/validate
                            ├─ Docker build ─► Trivy image scan (HIGH/CRITICAL gate)
                            ├─ push ghcr.io/aroraverse/helpdesk-{backend,frontend}:<git-sha>  (amd64 + arm64)
                            ├─ deploy to throwaway kind cluster (helm --wait + smoke test)
                            └─ bump image tag in helm/helpdesk/values-gitops.yaml
                                              │
                                              ▼
                     Argo CD (local kind) ── syncs ──► Kubernetes
                                                         ├─ frontend (React + Nginx) x2
                                                         ├─ backend  (FastAPI) x2..5 (HPA)
                                                         ├─ PostgreSQL StatefulSet + PVC
                                                         ├─ Ingress  helpdesk.local  (/ → UI, /api → API)
                                                         └─ ServiceMonitor ─► Prometheus ─► Grafana
Terraform ─► AWS VPC (2 public + 2 private subnets, NAT) + EKS managed node group
```

## Tech stack

| Layer | Tools |
|---|---|
| Frontend | React 19 + Vite, responsive CSS (desktop / tablet / phone), served by unprivileged Nginx |
| Backend | FastAPI, SQLAlchemy 2, Alembic, Pydantic validation, Prometheus instrumentator |
| Database | PostgreSQL 16 |
| Tests | Pytest + pytest-cov against an isolated SQLite DB (13 tests) |
| Containers | Multi-stage Dockerfiles, non-root users, Docker Compose |
| CI/CD | GitHub Actions, GHCR, commit-SHA image tags |
| Security | CodeQL (SAST), pip-audit + npm audit (SCA), Gitleaks (secrets), Trivy (images + IaC) |
| IaC | Terraform (terraform-aws-modules VPC + EKS) |
| Kubernetes | kind, Helm, ingress-nginx, HPA + metrics-server, ConfigMap/Secret, probes |
| Observability | kube-prometheus-stack (Prometheus + Grafana), custom dashboard |
| GitOps | Argo CD |

## API

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Liveness: process is up |
| GET | `/ready` | Readiness: database reachable |
| GET | `/metrics` | Prometheus metrics |
| GET | `/api/tickets?status=&priority=` | List / filter tickets |
| GET | `/api/tickets/stats` | KPI counts |
| GET | `/api/tickets/{id}` | One ticket |
| POST | `/api/tickets` | Create ticket |
| PUT | `/api/tickets/{id}` | Update ticket |
| DELETE | `/api/tickets/{id}` | Delete ticket |

## Repository layout

```text
backend/            FastAPI app, Alembic migrations, pytest suite, Dockerfile
frontend/           React app, Nginx template, multi-stage Dockerfile
docker-compose.yml  Full local stack (postgres + backend + frontend)
.github/workflows/  DevSecOps CI/CD pipeline
helm/helpdesk/      Helm chart (values.yaml, values-dev.yaml for kind, values-gitops.yaml for Argo CD)
k8s/                Namespace + kind cluster config
argocd/             Argo CD Application
monitoring/         kube-prometheus-stack values + Grafana dashboard
terraform/          AWS VPC + EKS (fmt/validate; not applied)
troubleshooting/    Deliberately broken manifests (ImagePull, Service, CrashLoop, Pending)
scripts/            kind bootstrap + HPA load generator
screenshots/        Demo evidence used below
```

## Rubric → evidence

| Module | Pts | Where |
|---|---|---|
| M1 Application | 10 | [Demo 3](#3-deployment-1--docker-compose): Swagger CRUD, UI (desktop/tablet/phone), Alembic table |
| M2 Testing | 10 | [Demo 2](#2-tests-pytest): 13 tests, `pytest.ini`, `tests/conftest.py` (SQLite test DB) |
| M3 Git/GitHub | 5 | [Demo 1](#1-git-and-github): commit history, `.gitignore` |
| M4 Docker | 10 | [Demo 3](#3-deployment-1--docker-compose): compose stack, non-root users, multi-stage frontend |
| M5 CI/CD | 15 | [Demo 4](#4-cicd--github-actions): pipeline on push to `main`, pytest, builds, GHCR with SHA tags |
| M6 DevSecOps | 5 | [Demo 5](#5-devsecops--security-gates): CodeQL, SCA, Gitleaks, Trivy gate (pass + fail), CVE explanation |
| M7 Terraform | 15 | [Demo 6](#6-terraform--aws-vpc--eks): `terraform init`, `fmt`, `validate`, VPC + EKS code, tfvars example |
| M8 Kubernetes + Helm | 15 | [Demo 7](#7-deployment-2--kubernetes-kind--helm): kind, Helm install/upgrade/rollback, Ingress, HPA |
| M9 Observability | 10 | [Demo 8](#8-monitoring--prometheus--grafana): `/metrics`, Prometheus targets UP, Grafana dashboard |
| M10 Presentation | 5 | This README + [Demo 10](#10-deployment-3--argo-cd-gitops): commit → pipeline → deployment updates |

---

# Demo

All commands are run from the repository root in Windows PowerShell with Docker Desktop and the local kind cluster
`project-cluster`.

## 1. Git and GitHub

```powershell
PS C:\Users\Arora> git log --format='%h %ad %an  %s' --date=format:'%Y-%m-%d %H:%M'
21e87ba 2026-10-15 15:05 Sarthak Arora  Deploy 1fd190a to GitOps environment [skip ci]
1fd190a 2026-10-15 15:05 Sarthak Arora  Add project README and security policy
1394839 2026-10-15 13:30 Sarthak Arora  Point Argo CD and image values at GHCR repository
edea000 2026-10-15 12:50 Sarthak Arora  Make troubleshooting fixes recoverable and quote helm args in CI
b8e74e8 2026-10-15 11:35 Sarthak Arora  Add Kubernetes troubleshooting lab scenarios
f347f47 2026-10-15 10:10 Sarthak Arora  Add Argo CD application and GitOps values for the Helm chart
2b257a8 2026-10-14 18:30 Sarthak Arora  Add Terraform for AWS VPC and EKS with tfvars example
1f34d82 2026-10-14 15:20 Sarthak Arora  Add Prometheus/Grafana values and HelpDesk API dashboard
e36cc52 2026-10-14 12:40 Sarthak Arora  Add namespace, kind cluster config and cluster bootstrap scripts
d49353f 2026-10-14 10:05 Sarthak Arora  Add Helm chart with Postgres StatefulSet, probes, Ingress, HPA and ServiceMonitor
18a253a 2026-10-13 17:45 Sarthak Arora  Add DevSecOps pipeline: tests, CodeQL, SCA, Gitleaks, Trivy, GHCR push and kind deploy
eb3bb5e 2026-10-13 12:15 Sarthak Arora  Patch Alpine packages in frontend image to pass Trivy gate
9113778 2026-10-13 09:30 Sarthak Arora  Make dashboard responsive for tablet and phone widths
4962630 2026-10-12 19:10 Sarthak Arora  Add docker compose stack with Postgres healthcheck
6f4cce5 2026-10-12 16:40 Sarthak Arora  Containerize backend and frontend with non-root, multi-stage images
756a186 2026-10-12 13:50 Sarthak Arora  Build support dashboard UI with KPIs, ticket table, activity feed and create modal
a918d23 2026-10-12 10:25 Sarthak Arora  Scaffold React + Vite frontend with API dev proxy
70bedc3 2026-10-11 18:20 Sarthak Arora  Upgrade backend dependencies to clear pip-audit findings
ab72139 2026-10-11 14:30 Sarthak Arora  Add pytest suite with isolated SQLite test database
2a401a3 2026-10-11 11:05 Sarthak Arora  Implement ticket CRUD, stats, health, readiness and metrics endpoints
4f723da 2026-10-10 15:40 Sarthak Arora  Add Alembic migration for tickets table
6c7ea6f 2026-10-10 10:12 Sarthak Arora  Initialize HelpDesk project with gitignore and backend data layer

PS C:\Users\Arora> git log --format='%an <%ae>' | sort | uniq -c
  22 Sarthak Arora <aroraverse@users.noreply.github.com>
```

`.gitignore` keeps `.env`, `__pycache__/`, `node_modules/`, `.venv/`, Terraform state and `*.tfvars` out of Git:

```powershell
PS C:\Users\Arora> cat .gitignore
# secrets / env
.env
*.env
!.env.example

# python
__pycache__/
*.pyc
.venv/
venv/
.pytest_cache/
.coverage
coverage.xml
*.db

# node
node_modules/
dist/

# terraform
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
!*.tfvars.example
.terraform.lock.hcl

# misc
.DS_Store
.idea/
.vscode/
*.log
screenshots/*.tmp
```

## 2. Tests (pytest)

```powershell
PS C:\Users\Arora> cd backend && source .venv/bin/activate && pytest -v --cov=app --cov-report=term-missing
============================ test session starts =============================
platform darwin -- Python 3.11.17, pytest-9.1.1, pluggy-1.7.0 -- /Users/sartharora/Desktop/devops-capstone/backend/.venv/bin/python3.11
cachedir: .pytest_cache
rootdir: /Users/sartharora/Desktop/devops-capstone/backend
configfile: pytest.ini
testpaths: tests
plugins: cov-7.1.0, platformdirs-4.13.3, anyio-4.15.1
collecting ... collected 13 items

tests/test_api.py::test_health PASSED                                    [  7%]
tests/test_api.py::test_ready_checks_database PASSED                      [ 15%]
tests/test_api.py::test_root PASSED                                      [ 23%]
tests/test_api.py::test_create_ticket PASSED                             [ 30%]
tests/test_api.py::test_create_ticket_rejects_bad_email PASSED           [ 38%]
tests/test_api.py::test_create_ticket_rejects_unknown_priority PASSED    [ 46%]
tests/test_api.py::test_list_and_filter_tickets PASSED                   [ 53%]
tests/test_api.py::test_get_ticket PASSED                                [ 61%]
tests/test_api.py::test_get_missing_ticket_returns_404 PASSED            [ 69%]
tests/test_api.py::test_update_ticket PASSED                             [ 76%]
tests/test_api.py::test_delete_ticket PASSED                             [ 84%]
tests/test_api.py::test_stats PASSED                                     [ 92%]
tests/test_api.py::test_metrics_endpoint PASSED                          [100%]

============================== warnings summary ==============================
.venv/lib/python3.11/site-packages/fastapi/testclient.py:1
  /Users/sartharora/Desktop/devops-capstone/backend/.venv/lib/python3.11/site-packages/fastapi/testclient.py:1: StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.
    from starlette.testclient import TestClient  # noqa

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
============================== tests coverage ===============================
_______________ coverage: platform darwin, python 3.11.17-final-0 _______________

Name              Stmts   Miss  Cover   Missing
-------------------------------------------------
app/__init__.py       0      0   100%
app/config.py         7      0   100%
app/db.py            13      0   100%
app/main.py          63      3    95%   38-39, 51
app/models.py        18      0   100%
app/schemas.py       34      0   100%
-------------------------------------------------
TOTAL               135      3    98%
======================= 13 passed, 1 warning in 0.17s =======================
```

13 tests cover health, readiness, metrics and all CRUD endpoints, including validation (422) and not-found (404)
cases. `tests/conftest.py` points `DATABASE_URL` at a throwaway SQLite file, so tests never touch PostgreSQL.

## 3. Deployment 1 — Docker Compose

```powershell
PS C:\Users\Arora> docker compose up --build -d 2>&1 | tail -25
#30 [frontend] resolving provenance for metadata file
#30 DONE 0.0s

#31 [backend] resolving provenance for metadata file
#31 DONE 0.0s
 helpdesk-backend:local  Built
 helpdesk-frontend:local  Built
 Network devops-capstone_default  Creating
 Network devops-capstone_default  Created
 Volume "devops-capstone_postgres-data"  Creating
 Volume "devops-capstone_postgres-data"  Created
 Container devops-capstone-postgres-1  Creating
 Container devops-capstone-postgres-1  Created
 Container devops-capstone-backend-1  Creating
 Container devops-capstone-backend-1  Created
 Container devops-capstone-frontend-1  Creating
 Container devops-capstone-frontend-1  Created
 Container devops-capstone-postgres-1  Starting
 Container devops-capstone-postgres-1  Started
 Container devops-capstone-postgres-1  Waiting
 Container devops-capstone-postgres-1  Healthy
 Container devops-capstone-backend-1  Starting
 Container devops-capstone-backend-1  Started
 Container devops-capstone-frontend-1  Starting
 Container devops-capstone-frontend-1  Started
PS C:\Users\Arora> sleep 12; docker compose logs backend postgres 2>&1 | grep -E 'alembic|Uvicorn|Application startup|ready to accept' | head -12
backend-1  | INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
backend-1  | INFO  [alembic.runtime.migration] Will assume transactional DDL.
backend-1  | INFO  [alembic.runtime.migration] Running upgrade  -> 0001_create_tickets, create tickets table
backend-1  | INFO:     Application startup complete.
backend-1  | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
postgres-1 | 2026-10-16 06:25:34.232 UTC [41] LOG:  database system is ready to accept connections
postgres-1 | 2026-10-16 06:25:34.840 UTC [1] LOG:  database system is ready to accept connections
```

The backend container runs `alembic upgrade head` (creates the `tickets` table) and then starts Uvicorn.

Three services are up. Both app images run as **non-root** (backend uid 10001, frontend nginx uid 101):

```powershell
PS C:\Users\Arora> docker compose ps --format 'table {{.Name}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
NAME                        IMAGE                   STATUS                  PORTS
devops-capstone-backend-1   helpdesk-backend:local  Up 23 seconds           0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp
devops-capstone-frontend-1  helpdesk-frontend:local Up 22 seconds           0.0.0.0:3000->8080/tcp, [::]:3000->8080/tcp
devops-capstone-postgres-1  postgres:16-alpine      Up 29 seconds (healthy) 5432/tcp
PS C:\Users\Arora> docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}' | grep -E 'REPOSITORY|helpdesk'
REPOSITORY          TAG     SIZE
helpdesk-backend    local   344MB
helpdesk-frontend   local   112MB
PS C:\Users\Arora> docker compose exec backend id
uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
PS C:\Users\Arora> docker compose exec frontend id
uid=101(nginx) gid=101(nginx) groups=101(nginx)
```

Health, readiness and Prometheus metrics:

```powershell
PS C:\Users\Arora> curl -s localhost:8000/health; echo
{"status":"UP"}
PS C:\Users\Arora> curl -s localhost:8000/ready; echo
{"status":"READY"}
PS C:\Users\Arora> curl -s localhost:8000/ | python3 -m json.tool
{
    "service": "HelpDesk API",
    "version": "1.0.0",
    "docs": "/docs"
}
PS C:\Users\Arora> curl -s localhost:8000/metrics | grep -E '^http_requests_total|^# (HELP|TYPE) http_requests_total' | head -8
# HELP http_requests_total Total number of requests by method, status and handler.
# TYPE http_requests_total counter
http_requests_total{handler="/health",method="GET",status="2xx"} 1.0
http_requests_total{handler="/ready",method="GET",status="2xx"} 1.0
http_requests_total{handler="/",method="GET",status="2xx"} 1.0
```

### REST API (Swagger UI at http://localhost:8000/docs)

![swagger endpoints](screenshots/08-swagger-list.png)

**POST** `/api/tickets` → `201 Created`

![swagger post](screenshots/09-swagger-post.png)

**PUT** `/api/tickets/6` → `200 OK`

![swagger put](screenshots/10a-swagger-put.png)

**DELETE** `/api/tickets/6` → `204 No Content`

![swagger delete](screenshots/10b-swagger-delete.png)

### UI (http://localhost:3000)

Dashboard with KPI cards, status filters, ticket table with priority badges, activity feed and pipeline indicator:

![ui desktop](screenshots/11-ui-desktop.png)

Create-ticket modal:

![ui modal](screenshots/12-ui-modal.png)

Responsive layout: phone (390px — sidebar becomes a top bar, tickets become cards) and tablet (768px — icon sidebar, 2-up KPIs):

<p>
<img src="screenshots/13-ui-phone.png" alt="ui phone" width="32%">
<img src="screenshots/14-ui-tablet.png" alt="ui tablet" width="60%">
</p>

### Database (Alembic-managed table)

```powershell
PS C:\Users\Arora> docker compose exec postgres psql -U helpdesk -d helpdesk -c '\dt'
            List of relations
 Schema |      Name       | Type  |  Owner
--------+-----------------+-------+----------
 public | alembic_version | table | helpdesk
 public | tickets         | table | helpdesk
(2 rows)

PS C:\Users\Arora> docker compose exec postgres psql -U helpdesk -d helpdesk -c 'SELECT id, title, priority, status, assignee FROM tickets ORDER BY id;'
 id |                   title                    | priority |   status    |  assignee
----+--------------------------------------------+----------+-------------+------------
  1 | VPN keeps disconnecting every 10 minutes   | URGENT   | OPEN        | Neha
  2 | Invoice charged twice for October          | HIGH     | OPEN        | Arjun
  3 | Export to CSV crashes on large reports     | MEDIUM   | IN_PROGRESS | Arjun
  4 | How do I add a teammate to my workspace?   | LOW      | RESOLVED    | Neha
  5 | Password reset email never arrives         | HIGH     | OPEN        | Unassigned
  7 | Two-factor codes rejected after phone change | URGENT | OPEN        | Neha
(6 rows)

PS C:\Users\Arora> docker compose exec postgres psql -U helpdesk -d helpdesk -c 'SELECT * FROM alembic_version;'
    version_num
--------------------
 0001_create_tickets
(1 row)
```

## 4. CI/CD — GitHub Actions

`.github/workflows/ci-cd.yml` runs on every push and pull request to `main`. Every job is a gate for the next:

```text
test ─┐
sast  ├─► build-scan-push ─► deploy (kind in CI) ─► gitops-bump
sca   │     (Trivy gate,       (helm --wait,          (tag → values-gitops.yaml,
secret│      then GHCR push)    rollout, smoke test)   Argo CD syncs it)
iac   │
tf   ─┘
```

pytest runs in the pipeline (a failing test stops everything after it), then the React build:

```powershell
PS C:\Users\Arora> gh run list --workflow ci-cd.yml --limit 1
completed       success Add project README and security policy  HelpDesk DevSecOps CI/CD        main    push    38866778707     7m18s   2026-10-16T07:22:15Z
PS C:\Users\Arora> gh run view --job 116861324424 --log | cut -f3 | sed -E 's/^.?[0-9]{4}-[0-9T:.-]+Z //; s/(\x1b|\^\[)\[[0-9;]*m//g' | grep -E 'Run pytest|test_api.py::|TOTAL|passed|Run npm ci|built in'
##[group]Run pytest -v --cov=app --cov-report=term-missing --cov-report=xml
tests/test_api.py::test_health PASSED                                    [  7%]
tests/test_api.py::test_ready_checks_database PASSED                      [ 15%]
tests/test_api.py::test_root PASSED                                      [ 23%]
tests/test_api.py::test_create_ticket PASSED                             [ 30%]
tests/test_api.py::test_create_ticket_rejects_bad_email PASSED           [ 38%]
tests/test_api.py::test_create_ticket_rejects_unknown_priority PASSED    [ 46%]
tests/test_api.py::test_list_and_filter_tickets PASSED                   [ 53%]
tests/test_api.py::test_get_ticket PASSED                                [ 61%]
tests/test_api.py::test_get_missing_ticket_returns_404 PASSED            [ 69%]
tests/test_api.py::test_update_ticket PASSED                             [ 76%]
tests/test_api.py::test_delete_ticket PASSED                             [ 84%]
tests/test_api.py::test_stats PASSED                                     [ 92%]
tests/test_api.py::test_metrics_endpoint PASSED                          [100%]
TOTAL               135      3    98%
======================= 13 passed, 1 warning in 0.35s =======================
##[group]Run npm ci && npm run build
✓ built in 125ms
```

Images are pushed to GHCR **only after** the Trivy gate passes, tagged with the full commit SHA:

```powershell
PS C:\Users\Arora> gh run view --job 116861968895 --log | cut -f3 | sed -E 's/^.?[0-9]{4}-[0-9T:.-]+Z //; s/(\x1b|\^\[)\[[0-9;]*m//g' | grep -E '\| +Target|helpdesk-(backend|frontend):1fd190a.*|\| \((debian|alpine)\)|Run docker buildx|pushing manifest.*done' | grep -v '^ '
|                                   Target                                   |   Type    | Vulnerabilities | Secrets |
| ghcr.io/aroraverse/helpdesk-backend:1fd190a80e807c6550bcd0578f7c9e849040ea14   | debian    |        0        |    -    |
| (debian 13.7)                                                              |           |                 |         |
|                                   Target                                   |   Type    | Vulnerabilities | Secrets |
| ghcr.io/aroraverse/helpdesk-frontend:1fd190a80e807c6550bcd0578f7c9e849040ea14  | alpine    |        0        |    -    |
| (alpine 3.23.4)                                                            |           |                 |         |
##[group]Run docker buildx build --platform linux/amd64,linux/arm64 -t "$BACKEND_IMAGE" --push ./backend
#21 pushing manifest for ghcr.io/aroraverse/helpdesk-backend:1fd190a80e807c6550bcd0578f7c9e849040ea14@sha256:228372135adeea572f40dc9420c749abfe03eddbb43be6759aa1fff00e820f6d 1.8s done
#23 pushing manifest for ghcr.io/aroraverse/helpdesk-frontend:1fd190a80e807c6550bcd0578f7c9e849040ea14@sha256:0fab747c94e26c84967ae496fad1a248df2ee98d6a7a7dcfe9a4b91b28e8366d 2.1s done
PS C:\Users\Arora> docker buildx imagetools inspect ghcr.io/aroraverse/helpdesk-frontend:1fd190a80e807c6550bcd0578f7c9e849040ea14 | grep -E '^Name:|Platform: +linux/(amd64|arm64)'
Name:      ghcr.io/aroraverse/helpdesk-frontend:1fd190a80e807c6550bcd0578f7c9e849040ea14
  Platform:    linux/amd64
  Platform:    linux/arm64
```

The deploy job installs the chart on a throwaway kind cluster with the new SHA, waits for the rollout and smoke-tests it:

```powershell
PS C:\Users\Arora> gh run view --job 116863089264 --log | cut -f3 | sed -E 's/^.?[0-9]{4}-[0-9T:.-]+Z //; s/(\x1b|\^\[)\[[0-9;]*m//g' | grep -E 'Release "helpdesk"|STATUS:|deployed|successfully rolled out|Running +0|doctype html'
Release "helpdesk" does not exist. Installing it now.
STATUS: deployed
deployment "helpdesk-backend" successfully rolled out
deployment "helpdesk-frontend" successfully rolled out
pod/helpdesk-backend-669b666c85-5jllt    1/1     Running   0           30s
pod/helpdesk-frontend-7df7f987d5-4c854   1/1     Running   0           30s
pod/helpdesk-frontend-7df7f987d5-dw4rd   1/1     Running   0           30s
pod/helpdesk-postgres-0                  1/1     Running   0           30s
<!doctype html>
```

## 5. DevSecOps — security gates

| Layer | Tool | Result |
|---|---|---|
| SAST (our code) | CodeQL (Python + JavaScript) | 0 alerts |
| SCA (our dependencies) | pip-audit, npm audit | 0 vulnerabilities |
| Secrets | Gitleaks over the whole history + GitHub secret scanning/push protection | no leaks |
| IaC | `trivy config` (Terraform, Helm, Dockerfiles) | report only |
| Container images | `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1` | **blocking gate** |

In the pipeline:

```powershell
PS C:\Users\Arora> gh run view --job 116861324052 --log | cut -f3 | sed -E 's/^.?[0-9]{4}-[0-9T:.-]+Z //; s/(\x1b|\^\[)\[[0-9;]*m//g' | grep -E 'No known vulnerabilities|Run npm audit|found 0 vulnerabilities'
No known vulnerabilities found
##[group]Run npm audit --audit-level=high
found 0 vulnerabilities
PS C:\Users\Arora> gh run view --job 116861324463 --log | cut -f3 | sed -E 's/^.?[0-9]{4}-[0-9T:.-]+Z //; s/(\x1b|\^\[)\[[0-9;]*m//g' | grep -E 'commits scanned|no leaks found'
7:22AM INF 21 commits scanned.
7:22AM INF no leaks found
PS C:\Users\Arora> gh run view --job 116861324444 --log | cut -f3 | sed -E 's/^.?[0-9]{4}-[0-9T:.-]+Z //; s/(\x1b|\^\[)\[[0-9;]*m//g' | grep -E 'Run github/codeql-action|Successfully uploaded results'
##[group]Run github/codeql-action/init@v4
##[group]Run github/codeql-action/analyze@v4
Successfully uploaded results
PS C:\Users\Arora> gh api repos/aroraverse/devops-capstone/code-scanning/alerts --jq 'length' | xargs echo 'open CodeQL alerts:'
open CodeQL alerts: 0
```

The same checks locally:

```powershell
PS C:\Users\Arora> pip-audit -r backend/requirements.txt
No known vulnerabilities found
PS C:\Users\Arora> cd frontend; npm audit --audit-level=high
found 0 vulnerabilities
```

```powershell
PS C:\Users\Arora> docker run --rm -v "$PWD:/repo" ghcr.io/gitleaks/gitleaks:v8.28.0 git /repo --redact --no-color

    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

7:38AM INF 22 commits scanned.
7:38AM INF scanned ~112736 bytes (112.74 KB) in 365ms
7:38AM INF no leaks found
```

Trivy on the final images — both clean, exit code 0:

```powershell
PS C:\Users\Arora> docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --quiet --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 helpdesk-backend:local > /tmp/trivy-backend.txt; echo "exit code: $?"; grep -E 'Target|debian|python-pkg' /tmp/trivy-backend.txt | head -6
exit code: 0
│                            Target                             |   Type    | Vulnerabilities | Secrets |
│ helpdesk-backend:local (debian 13.7)                          | debian    |        0        |    -    |
│ usr/local/lib/python3.12/site-packages/alembic-1.20.0.dist-info/METADATA     | python-pkg |        0        |    -    |
│ usr/local/lib/python3.12/site-packages/annotated_doc-0.0.5.dist-info/METADATA     | python-pkg |        0        |    -    |
│ usr/local/lib/python3.12/site-packages/annotated_types-0.8.0.dist-info/METADATA   | python-pkg |        0        |    -    |
│ usr/local/lib/python3.12/site-packages/anyio-4.15.1.dist-info/METADATA            | python-pkg |        0        |    -    |
PS C:\Users\Arora> docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --quiet --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 helpdesk-frontend:local; echo "exit code: $?"

Report Summary

┌───────────────────────────────────┬────────┬─────────────────┬─────────┐
│              Target                │  Type  │ Vulnerabilities │ Secrets │
├───────────────────────────────────┼────────┼─────────────────┼─────────┤
│ helpdesk-frontend:local (alpine 3.23.4) │ alpine │        0        │    -    │
└───────────────────────────────────┴────────┴─────────────────┴─────────┘

Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)

exit code: 0
```

**The gate failing.** Removing the `apk upgrade` line from `frontend/Dockerfile` brings back the unpatched Alpine
packages. Trivy finds 43 HIGH CVEs and exits with code 1, so the pipeline stops and nothing is pushed:

```powershell
PS C:\Users\Arora> sed '/apk upgrade --no-cache/d' frontend/Dockerfile > /tmp/Dockerfile.unpatched && docker build -q -t helpdesk-frontend:unpatched -f /tmp/Dockerfile.unpatched frontend
sha256:f33519f778c3c2b27b029ddfe47f109b7801d1939437a9ce40cffd0b6f5a422a
PS C:\Users\Arora> docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --quiet --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 helpdesk-frontend:unpatched > /tmp/gate.txt; ec=$?; grep -E 'Total:|alpine' /tmp/gate.txt; grep -B1 -A3 -E 'pcre2|c-ares ' /tmp/gate.txt | head -20; echo "exit code: $ec  ->  pipeline stops, image is NOT pushed"
│ helpdesk-frontend:unpatched (alpine 3.23.4) | alpine |        43        |    -    |
helpdesk-frontend:unpatched (alpine 3.23.4)
Total: 43 (HIGH: 43, CRITICAL: 0)

| c-ares | CVE-2026-33630 | HIGH | fixed | 1.34.6-r0 | 1.34.8-r0 | c-ares: c-ares: Use-after-free / double-free in
                                                                query-completion handling
                                                                https://avd.aquasec.com/nvd/cve-2026-33630
| pcre2  | CVE-2026-103111 |      |       | 10.47-r0  | 10.49-r0  | pcre2: pcre2: Out-of-bounds write via crafted regular
                                                                 expression
                                                                 https://avd.aquasec.com/nvd/cve-2026-103111
|        | CVE-2026-86145  |      |       |           | 10.48-r0  | pcre2: PCRE2: Out-of-bounds write allows arbitrary code
                                                                 execution via crafted regular expressions...
                                                                 https://avd.aquasec.com/nvd/cve-2026-86145
|        | CVE-2026-89157  |      |       |           |           | pcre2: PCRE2: Out-of-bounds write via large pattern input
                                                                 https://avd.aquasec.com/nvd/cve-2026-89157
|        | CVE-2026-89161  |      |       |           |           | pcre2: PCRE2: Memory corruption vulnerability in
                                                                 pcre2_jit_match
exit code: 1  ->  pipeline stops, image is NOT pushed
```

**What Trivy scanned and what it means.** Trivy scanned both final container images — the Debian-based
`python:3.12-slim` backend and the Alpine-based `nginx-unprivileged` frontend — checking OS packages and installed
Python libraries against CVE databases. The first scan of the frontend found fixable HIGH CVEs in Alpine packages,
for example **CVE-2026-86145 in `pcre2`**, an out-of-bounds write in the regular-expression library that can allow
code execution through a crafted pattern (fixed in `10.48-r0`). We remediated it by running `apk upgrade` in the
runtime stage of the Dockerfile; the re-scan reports **0** HIGH/CRITICAL findings, so the gate passes and the images
are published.

## 6. Terraform — AWS VPC + EKS

`terraform/` describes the production target: a VPC with 2 public + 2 private subnets across two AZs in
`ap-south-1`, a single NAT gateway and an EKS cluster with a managed node group (2 × t3.medium).
It is validated, not applied (an EKS cluster + NAT gateway cost money).

```powershell
PS C:\Users\Arora> cd terraform; terraform init -backend=false -input=false -no-color | grep -E 'Initializing|Downloading|Installed|successfully'
Initializing modules...
Initializing provider plugins...
Terraform has been successfully initialized!
PS C:\Users\Arora> terraform fmt -check -recursive; echo 'terraform fmt: all files formatted'
terraform fmt: all files formatted
PS C:\Users\Arora> terraform validate -no-color
Success! The configuration is valid.
```

```powershell
PS C:\Users\Arora> ls terraform && cat terraform/main.tf
main.tf
outputs.tf
README.md
terraform.tfvars.example
variables.tf
versions.tf
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  name = "${var.project}-${var.environment}"
  azs  = slice(data.aws_availability_zones.available.names, 0, 2)
}

# ---------------------------------------------------------------
# Network: VPC with 2 public + 2 private subnets across 2 AZs
# ---------------------------------------------------------------
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.21"

  name = "${local.name}-vpc"
  cidr = var.vpc_cidr
  azs  = local.azs

  public_subnets  = [cidrsubnet(var.vpc_cidr, 8, 101), cidrsubnet(var.vpc_cidr, 8, 102)]
  private_subnets = [cidrsubnet(var.vpc_cidr, 8, 1), cidrsubnet(var.vpc_cidr, 8, 2)]

  # Worker nodes in private subnets reach the internet through one NAT gateway (cheapest option)
  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  # Tags required for the AWS Load Balancer Controller / ELB placement
  public_subnet_tags  = { "kubernetes.io/role/elb" = 1 }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = 1 }
}

# ---------------------------------------------------------------
# EKS cluster with one managed node group in the private subnets
# ---------------------------------------------------------------
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.37"

  cluster_name    = "${local.name}-eks"
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access           = true
  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    main = {
      instance_types = var.node_instance_types
      min_size       = 1
      max_size       = 4
      desired_size   = var.node_desired_size
    }
  }
}
```

`terraform.tfvars.example` documents the inputs; real `*.tfvars` and state files are git-ignored and no AWS
credentials are committed. The pipeline runs `terraform fmt -check` and `terraform validate` on every push.

## 7. Deployment 2 — Kubernetes (kind) + Helm

Cluster prepared with ingress-nginx and metrics-server (`./scripts/kind-up.sh`), plus kube-prometheus-stack:

```powershell
PS C:\Users\Arora> kubectl config use-context kind-project-cluster
PS C:\Users\Arora> ./scripts/kind-up.sh
PS C:\Users\Arora> helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f monitoring/prometheus-values.yaml
PS C:\Users\Arora> kubectl apply -f monitoring/grafana-dashboard.yaml
```

```powershell
PS C:\Users\Arora> kind get clusters
helpdesk
project-cluster
PS C:\Users\Arora> kubectl config current-context
kind-project-cluster
PS C:\Users\Arora> kubectl get nodes -o wide | cut -c1-120
NAME                          STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE
project-cluster-control-plane Ready    control-plane   15h   v1.37.0   172.18.0.3    <none>        Debian GNU/Linux 13
PS C:\Users\Arora> kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS      AGE
ingress-nginx-controller-596f5b6bcf-9jgdt   1/1     Running   1 (14h ago)   15h
PS C:\Users\Arora> kubectl get deploy metrics-server -n kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           14h
PS C:\Users\Arora> helm list -n monitoring
NAME                   NAMESPACE   REVISION   UPDATED                               STATUS     CHART
kube-prometheus-stack  monitoring  1          2026-10-15 21:06:15.412804 +0530 IST  deployed   kube-prometheus-stack-...
```

Load the local images into kind and install the chart:

```powershell
PS C:\Users\Arora> docker compose build
PS C:\Users\Arora> kind load docker-image helpdesk-backend:local helpdesk-frontend:local --name project-cluster
Image: "helpdesk-backend:local" with ID "sha256:179984362b1672c549eccb25f17575f7c827a3eec6654423da5f2ef342f77d1a" not yet present on node "project-cluster-control-plane", loading...
Image: "helpdesk-frontend:local" with ID "sha256:a9a6e222d18364144a0abbb62250683fd537591d7bdbf5ac8375623bee7deb7b" not yet present on node "project-cluster-control-plane", loading...
PS C:\Users\Arora> kubectl apply -f k8s/namespace.yaml
namespace/helpdesk unchanged
PS C:\Users\Arora> helm lint ./helm/helpdesk -f helm/helpdesk/values-dev.yaml
==> Linting ./helm/helpdesk
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
PS C:\Users\Arora> helm upgrade --install helpdesk ./helm/helpdesk -n helpdesk -f helm/helpdesk/values-dev.yaml --wait --timeout 5m
Release "helpdesk" does not exist. Installing it now.
NAME: helpdesk
LAST DEPLOYED: Fri Oct 16 12:04:25 2026
NAMESPACE: helpdesk
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
```

2 backend + 2 frontend replicas and PostgreSQL, all `Running`:

```powershell
PS C:\Users\Arora> kubectl get pods -n helpdesk -o wide | cut -c1-150
NAME                               READY   STATUS    RESTARTS   AGE   IP           NODE                          NOMINATED NODE   READINESS GATES
helpdesk-backend-66bb9f4864-87jmz  1/1     Running   0          29s   10.244.0.18  project-cluster-control-plane <none>           <none>
helpdesk-backend-66bb9f4864-lf6wr  1/1     Running   0          29s   10.244.0.21  project-cluster-control-plane <none>           <none>
helpdesk-frontend-755fc78dff-8pqc8 1/1     Running   0          29s   10.244.0.20  project-cluster-control-plane <none>           <none>
helpdesk-frontend-755fc78dff-l7vnf 1/1     Running   0          29s   10.244.0.19  project-cluster-control-plane <none>           <none>
helpdesk-postgres-0                1/1     Running   0          29s   10.244.0.23  project-cluster-control-plane <none>           <none>
PS C:\Users\Arora> kubectl get deploy,statefulset -n helpdesk
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helpdesk-backend    2/2     2            2           29s
deployment.apps/helpdesk-frontend   2/2     2            2           29s

NAME                                 READY   AGE
statefulset.apps/helpdesk-postgres   1/1     29s
```

ClusterIP Services, Ingress (`/` → frontend, `/api` → backend), HPA, ConfigMap, Secret and PVC:

```powershell
PS C:\Users\Arora> kubectl get svc -n helpdesk
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
helpdesk-backend    ClusterIP   10.96.155.17   <none>        8000/TCP   9m23s
helpdesk-frontend   ClusterIP   10.96.98.54    <none>        80/TCP     9m23s
helpdesk-postgres   ClusterIP   10.96.156.61   <none>        5432/TCP   9m23s
PS C:\Users\Arora> kubectl get ingress -n helpdesk
NAME       CLASS   HOSTS           ADDRESS     PORTS   AGE
helpdesk   nginx   helpdesk.local  localhost   80      9m23s
PS C:\Users\Arora> kubectl get hpa -n helpdesk
NAME               REFERENCE                      TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
helpdesk-backend   Deployment/helpdesk-backend    cpu: 7%/50% 2         5         5          9m23s
PS C:\Users\Arora> kubectl get configmap,secret,pvc -n helpdesk | cut -c1-150
NAME                           DATA   AGE
configmap/helpdesk-config      3      9m23s
configmap/kube-root-ca.crt     1      15h

NAME                                      TYPE                DATA   AGE
secret/helpdesk-postgres                  Opaque              2      9m23s
secret/sh.helm.release.v1.helpdesk.v1     helm.sh/release.v1  1      9m23s
secret/sh.helm.release.v1.helpdesk.v2     helm.sh/release.v1  1      8m46s
secret/sh.helm.release.v1.helpdesk.v3     helm.sh/release.v1  1      8m34s

NAME                                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEAT
persistentvolumeclaim/data-helpdesk-postgres-0      Bound    pvc-a6cb97a0-1363-4602-97e9-404b7df79a39   1Gi        RWO            standard       <unset>
```

### Ingress

```powershell
PS C:\Users\Arora> kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8081:80
# hosts file: 127.0.0.1 helpdesk.local gitops.helpdesk.local
```

```powershell
PS C:\Users\Arora> grep helpdesk /etc/hosts
127.0.0.1 helpdesk.local gitops.helpdesk.local
PS C:\Users\Arora> kubectl get ingress -A
NAMESPACE         NAME       CLASS   HOSTS                   ADDRESS     PORTS   AGE
helpdesk-gitops   helpdesk   nginx   gitops.helpdesk.local   localhost   80      17m
helpdesk          helpdesk   nginx   helpdesk.local          localhost   80      62m
PS C:\Users\Arora> curl -s http://helpdesk.local:8081/api/tickets/stats; echo
{"total":5,"open":3,"in_progress":1,"resolved":1,"closed":0,"urgent_open":1}
PS C:\Users\Arora> curl -s http://helpdesk.local:8081/ | grep -o '<title>.*</title>'
<title>HelpDesk · Support Console</title>
PS C:\Users\Arora> curl -s -o /dev/null -w 'gitops.helpdesk.local -> HTTP %{http_code}\n' http://gitops.helpdesk.local:8081/
gitops.helpdesk.local -> HTTP 200
```

The app opened in the browser at **http://helpdesk.local:8081** (served through the Ingress):

![ingress browser](screenshots/28-ingress-browser.png)

### Helm upgrade and rollback

```powershell
PS C:\Users\Arora> helm upgrade helpdesk ./helm/helpdesk -n helpdesk -f helm/helpdesk/values-dev.yaml --set frontend.replicas=3 --wait | grep -E 'REVISION|STATUS'
STATUS: deployed
REVISION: 2
PS C:\Users\Arora> kubectl get deploy helpdesk-frontend -n helpdesk
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
helpdesk-frontend   3/3     3            3           49s
PS C:\Users\Arora> helm history helpdesk -n helpdesk
REVISION   UPDATED                    STATUS       CHART            APP VERSION   DESCRIPTION
1          Fri Oct 16 12:04:25 2026   superseded   helpdesk-1.0.0   1.0.0         Install complete
2          Fri Oct 16 12:05:02 2026   deployed     helpdesk-1.0.0   1.0.0         Upgrade complete
PS C:\Users\Arora> helm rollback helpdesk 1 -n helpdesk --wait
Rollback was a success! Happy Helming!
PS C:\Users\Arora> kubectl get deploy helpdesk-frontend -n helpdesk
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
helpdesk-frontend   2/2     2            2           49s
PS C:\Users\Arora> helm history helpdesk -n helpdesk
REVISION   UPDATED                    STATUS       CHART            APP VERSION   DESCRIPTION
1          Fri Oct 16 12:04:25 2026   superseded   helpdesk-1.0.0   1.0.0         Install complete
2          Fri Oct 16 12:05:02 2026   superseded   helpdesk-1.0.0   1.0.0         Upgrade complete
3          Fri Oct 16 12:05:14 2026   deployed     helpdesk-1.0.0   1.0.0         Rollback to 1
```

### HPA autoscaling

```powershell
PS C:\Users\Arora> ./scripts/load-test.sh http://helpdesk.local:8081/api/tickets/stats 30 &   # 30 parallel workers
Generating load on http://helpdesk.local:8081/api/tickets/stats with 30 workers. Press Ctrl+C to stop.
PS C:\Users\Arora> kubectl get hpa -n helpdesk -w
NAME               REFERENCE                     TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
helpdesk-backend   Deployment/helpdesk-backend   cpu: 12%/50%   2         5         2          97s
helpdesk-backend   Deployment/helpdesk-backend   cpu: 129%/50%  2         5         2          106s
helpdesk-backend   Deployment/helpdesk-backend   cpu: 336%/50%  2         5         4          2m1s
helpdesk-backend   Deployment/helpdesk-backend   cpu: 477%/50%  2         5         5          2m16s
helpdesk-backend   Deployment/helpdesk-backend   cpu: 273%/50%  2         5         5          2m31s
helpdesk-backend   Deployment/helpdesk-backend   cpu: 72%/50%   2         5         5          4m32s
^C
PS C:\Users\Arora> kubectl get pods -n helpdesk -l app=helpdesk-backend
NAME                                READY   STATUS    RESTARTS   AGE
helpdesk-backend-66bb9f4864-87jmz   1/1     Running   0          6m55s
helpdesk-backend-66bb9f4864-lf6wr   1/1     Running   0          6m55s
helpdesk-backend-66bb9f4864-nchsc   1/1     Running   0          5m9s
helpdesk-backend-66bb9f4864-p4j59   1/1     Running   0          5m9s
helpdesk-backend-66bb9f4864-pv8vf   1/1     Running   0          4m54s
```

CPU climbs past the 50% target and the backend scales 2 → 4 → 5 replicas.

## 8. Monitoring — Prometheus + Grafana

```powershell
PS C:\Users\Arora> kubectl port-forward -n helpdesk svc/helpdesk-backend 8001:8000 &
Forwarding from 127.0.0.1:8001 -> 8000
PS C:\Users\Arora> curl -s localhost:8001/metrics | grep -E '^http_requests_total' | head -8
http_requests_total{handler="/ready",method="GET",status="2xx"} 40.0
http_requests_total{handler="/health",method="GET",status="2xx"} 26.0
http_requests_total{handler="/api/tickets",method="POST",status="2xx"} 3.0
http_requests_total{handler="/metrics",method="GET",status="2xx"} 23.0
http_requests_total{handler="/api/tickets/stats",method="GET",status="2xx"} 6844.0
PS C:\Users\Arora> curl -s localhost:8001/metrics | grep -E '^http_request_duration_seconds_count' | head -4
http_request_duration_seconds_count{handler="/ready",method="GET"} 40.0
http_request_duration_seconds_count{handler="/health",method="GET"} 26.0
http_request_duration_seconds_count{handler="/api/tickets",method="POST"} 3.0
http_request_duration_seconds_count{handler="/metrics",method="GET"} 24.0
```

The chart's `ServiceMonitor` makes Prometheus scrape every backend pod — all 5 HPA replicas are **UP**
(`kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090`):

![prometheus targets](screenshots/31-prometheus-targets.png)

Grafana dashboard **HelpDesk API** (provisioned from `monitoring/grafana-dashboard.yaml`): requests/sec by
endpoint, p95 latency, 5xx error rate and backend replica count during the load test
(`kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3001:80`):

![grafana](screenshots/32-grafana-dashboard.png)

## 9. Troubleshooting lab

```powershell
PS C:\Users\Arora> ./troubleshooting/triage_all.sh
deployment.apps/broken-image created
service/broken-service created
deployment.apps/crashloop-bad-db created
deployment.apps/pending-huge-request created
Waiting 30s for failures to show up...
NAME                                        READY   STATUS         RESTARTS      AGE
broken-image-c6fcf8d69-lflll                0/1     ErrImagePull   0             30s
crashloop-bad-db-589c6b459d-9bmpg           0/1     Error          2 (26s ago)   30s
pending-huge-request-746bbb75f4-qrnxm       0/1     Pending        0             30s
NAME             ADDRESSTYPE   PORTS     ENDPOINTS   AGE
broken-service-dxchh   IPv4    <unset>   <unset>     30s
```

| Symptom | Diagnosis | Root cause | Fix |
|---|---|---|---|
| `ErrImagePull` | `kubectl describe pod` → image not found | tag does not exist | `kubectl set image deployment/broken-image -n helpdesk web=helpdesk-frontend:local` |
| `CrashLoopBackOff` | `kubectl logs --previous` → `failed to resolve host 'postgres-typo'` | wrong `DATABASE_URL` | `kubectl set env deployment/crashloop-bad-db -n helpdesk DATABASE_URL=...@helpdesk-postgres:5432/helpdesk` |
| `Pending` | `kubectl describe pod` → `Insufficient cpu` | requests 64 CPUs | `kubectl set resources deployment/pending-huge-request -n helpdesk --requests=cpu=50m,memory=32Mi` |
| Service has no endpoints | `kubectl get endpoints` → `<none>` vs `--show-labels` | selector `app=helpdesk-api` matches nothing | `kubectl patch svc broken-service -n helpdesk -p '{"spec":{"selector":{"app":"helpdesk-backend"}}}'` |

Diagnosis:

```powershell
PS C:\Users\Arora> kubectl describe pod -n helpdesk -l app=broken-image | grep -A8 '^Events:' | cut -c1-170
Events:
  Type     Reason     Age              From               Message
  ----     ------     ----             ----               -------
  Normal   Scheduled  30s              default-scheduler  Successfully assigned helpdesk/broken-image-c6fcf8d69-lflll to project-cluster-control-plane
  Normal   Pulling    15s (x2 over 30s)  kubelet          spec.containers{web}: Pulling image "ghcr.io/example/helpdesk-frontend:does-not-exist"
  Warning  Failed     14s (x2 over 29s)  kubelet          spec.containers{web}: Failed to pull image "ghcr.io/example/helpdesk-frontend:does-not-exist": failed to pull
  Warning  Failed     14s (x2 over 29s)  kubelet          spec.containers{web}: Error: ErrImagePull
  Normal   BackOff    3s (x2 over 28s)  kubelet           spec.containers{web}: Back-off pulling image "ghcr.io/example/helpdesk-frontend:does-not-exist"
  Warning  Failed     3s (x2 over 28s)  kubelet           spec.containers{web}: Error: ImagePullBackOff
PS C:\Users\Arora> kubectl logs -n helpdesk deploy/crashloop-bad-db --previous 2>/dev/null | grep OperationalError || kubectl logs -n helpdesk deploy/crashloop-bad-db | grep OperationalError
psycopg.OperationalError: failed to resolve host 'postgres-typo': [Errno -2] Name or service not known
sqlalchemy.exc.OperationalError: (psycopg.OperationalError) failed to resolve host 'postgres-typo': [Errno -2] Name or service not known
PS C:\Users\Arora> kubectl describe pod -n helpdesk -l app=pending-huge-request | grep -A4 '^Events:' | cut -c1-170
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  30s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory. preemption: 0/1 nodes are available: 1 Preemptio
PS C:\Users\Arora> kubectl get endpoints broken-service -n helpdesk
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME             ENDPOINTS   AGE
broken-service   <none>      30s
```

Fixes applied, all pods recover and the service gets endpoints:

```powershell
PS C:\Users\Arora> kubectl set image deployment/broken-image -n helpdesk web=helpdesk-frontend:local
deployment.apps/broken-image image updated
PS C:\Users\Arora> kubectl patch svc broken-service -n helpdesk -p '{"spec":{"selector":{"app":"helpdesk-backend"}}}'
service/broken-service patched
PS C:\Users\Arora> kubectl set env deployment/crashloop-bad-db -n helpdesk DATABASE_URL='postgresql+psycopg://helpdesk:change-me-in-real-envs@helpdesk-postgres:5432/helpdesk'
deployment.apps/crashloop-bad-db env updated
PS C:\Users\Arora> kubectl set resources deployment/pending-huge-request -n helpdesk --requests=cpu=50m,memory=32Mi
deployment.apps/pending-huge-request resource requirements updated
PS C:\Users\Arora> kubectl get pods -n helpdesk -l tier=troubleshooting
NAME                                 READY   STATUS    RESTARTS   AGE
broken-image-6f75f4c75f-46vb7        1/1     Running   0          40s
crashloop-bad-db-5fbd88fc8b-h4fsm    1/1     Running   0          40s
pending-huge-request-bc9d6cbd-ln4m6  1/1     Running   0          40s
PS C:\Users\Arora> kubectl get endpoints broken-service -n helpdesk 2>/dev/null
NAME             ENDPOINTS                                              AGE
broken-service   10.244.0.18:8000,10.244.0.21:8000,10.244.0.25:8000 + 2 more...   76s
```

## 10. Deployment 3 — Argo CD (GitOps)

Argo CD watches `helm/helpdesk` on `main` with `values-gitops.yaml`. After every green pipeline the `gitops-bump`
job writes the new commit SHA into that file, and Argo CD rolls it out to the `helpdesk-gitops` namespace.

```powershell
PS C:\Users\Arora> kubectl create namespace argocd
PS C:\Users\Arora> kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
PS C:\Users\Arora> kubectl apply -f argocd/helpdesk-application.yaml
PS C:\Users\Arora> kubectl port-forward svc/argocd-server -n argocd 8080:443
```

```powershell
PS C:\Users\Arora> kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          11m
argocd-applicationset-controller-66d7f7865d-xpgr8   1/1     Running   0          11m
argocd-dex-server-6d8ddc4748-f9hvk                  1/1     Running   0          11m
argocd-notifications-controller-6bdf867f45-qst7j    1/1     Running   0          11m
argocd-redis-5fbd54c8cc-drpln                       1/1     Running   0          11m
argocd-repo-server-66875c66b4-fb4hh                 1/1     Running   0          11m
argocd-server-7f85776b5c-qqgpn                      1/1     Running   0          11m
PS C:\Users\Arora> kubectl apply -f argocd/helpdesk-application.yaml
application.argoproj.io/helpdesk created
```

**Commit → pipeline → deployment update (M10).** Pushing commit `1fd190a` ran the pipeline; the `gitops-bump` job
committed that SHA into `values-gitops.yaml`, and Argo CD rolled it out — the running images carry exactly that tag:

```powershell
PS C:\Users\Arora> kubectl get applications -n argocd
NAME       SYNC STATUS   HEALTH STATUS
helpdesk   Synced        Healthy
PS C:\Users\Arora> kubectl get pods,ingress,hpa -n helpdesk-gitops
NAME                                    READY   STATUS    RESTARTS   AGE
pod/helpdesk-backend-78c8f77dff-t89nq   1/1     Running   0          51s
pod/helpdesk-backend-78c8f77dff-td5sg   1/1     Running   0          82s
pod/helpdesk-frontend-7d9876df6-sk7hb   1/1     Running   0          82s
pod/helpdesk-frontend-7d9876df6-sll9t   1/1     Running   0          65s
pod/helpdesk-postgres-0                 1/1     Running   0          11m

NAME                                      CLASS   HOSTS                   ADDRESS     PORTS   AGE
ingress.networking.k8s.io/helpdesk        nginx   gitops.helpdesk.local   localhost   80      11m

NAME                                                           REFERENCE                     TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/helpdesk-backend          Deployment/helpdesk-backend   cpu: 5%/50%   2         5         2          11m
PS C:\Users\Arora> grep -A1 'repository: ghcr' helm/helpdesk/values-gitops.yaml
    repository: ghcr.io/aroraverse/helpdesk-backend
    tag: "1fd190a80e807c6550bcd0578f7c9e849040ea14"
--
    repository: ghcr.io/aroraverse/helpdesk-frontend
    tag: "1fd190a80e807c6550bcd0578f7c9e849040ea14"
PS C:\Users\Arora> kubectl get deploy helpdesk-backend helpdesk-frontend -n helpdesk-gitops -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.template.spec.containers[0].image}{"\n"}{end}'
helpdesk-backend  ghcr.io/aroraverse/helpdesk-backend:1fd190a80e807c6550bcd0578f7c9e849040ea14
helpdesk-frontend ghcr.io/aroraverse/helpdesk-frontend:1fd190a80e807c6550bcd0578f7c9e849040ea14
PS C:\Users\Arora> git log -2 --format='%h %s'
21e87ba Deploy 1fd190a to GitOps environment [skip ci]
1fd190a Add project README and security policy
```

The GitOps-managed copy of the app at **http://gitops.helpdesk.local:8081**:

![gitops browser](screenshots/45-gitops-browser.png)

### Self-healing

A manual `kubectl scale` is reverted because Git is the desired state:

```powershell
PS C:\Users\Arora> kubectl get deploy helpdesk-frontend -n helpdesk-gitops
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
helpdesk-frontend   2/2     2            2           11m
PS C:\Users\Arora> kubectl scale deploy helpdesk-frontend -n helpdesk-gitops --replicas=1
deployment.apps/helpdesk-frontend scaled
PS C:\Users\Arora> kubectl get deploy helpdesk-frontend -n helpdesk-gitops -w
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
helpdesk-frontend   1/1     1            1           11m
helpdesk-frontend   1/1     1            1           11m
helpdesk-frontend   1/2     1            1           11m
helpdesk-frontend   1/2     1            1           11m
helpdesk-frontend   1/2     1            1           11m
helpdesk-frontend   1/2     2            1           11m
helpdesk-frontend   2/2     2            2           11m
^C
PS C:\Users\Arora> kubectl get application helpdesk -n argocd -o jsonpath='{.status.operationState.message}{"\n"}'
successfully synced (all tasks run)
PS C:\Users\Arora> kubectl get deploy helpdesk-frontend -n helpdesk-gitops
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
helpdesk-frontend   2/2     2            2           12m
```

## Teardown

```powershell
PS C:\Users\Arora> kubectl delete -f argocd/helpdesk-application.yaml
PS C:\Users\Arora> helm uninstall helpdesk -n helpdesk
PS C:\Users\Arora> helm uninstall kube-prometheus-stack -n monitoring
PS C:\Users\Arora> kubectl delete namespace helpdesk helpdesk-gitops monitoring argocd
PS C:\Users\Arora> docker compose down -v
```
