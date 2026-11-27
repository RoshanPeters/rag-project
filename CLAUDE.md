# RAG Project — Context & Plan

## Who I am & how I want to work
- **Goal:** learn the concepts thoroughly, not just ship a demo.
- **Level:** comfortable with Python; **complete beginner** with the terminal/shell, git/GitHub, Docker and AWS. Explain tooling step by step (exact commands + what each part does + what I should see). Don't assume shell knowledge (PATH, sourcing, current directory, etc.).
- **Working style (for any AI assistant helping me):**
  - Teach the concept first, then give me a **spec + hints**, not full solutions. I write the code; you review it like a senior engineer reviewing a PR. (For tiny Phase 0 files, a reference version is OK *after* I've tried.)
  - Build from primitives before frameworks — **no LangChain/LlamaIndex initially**. Later, rebuild one piece with a framework to compare.
  - Introduce Docker and AWS gradually, only when the project needs them.
- **Learning loop per phase:** Concept → Build → Break it on purpose → Measure (once evals exist) → Explain it back (checkpoint questions).
- I keep a learning log in `docs/learning-log.md` (what I tried, what surprised me, what I'd do differently).
- **Corpus:** still to choose — documents I know well (my notes, docs for a tool I use daily, PDFs from my field) so I can judge answer quality and write good evals.

## Stack
- Python 3.12 + FastAPI, **uv** for dependencies/envs/lockfile
- Postgres + **pgvector** (vectors + metadata in one DB; maps to AWS RDS)
- LLM API (Claude / OpenAI / Bedrock) + an embedding model
- Docker Compose locally, Terraform for AWS, GitHub Actions for CI/CD
- ruff (lint/format), pytest

## Roadmap
**Phase 0 — Foundations.** Repo, FastAPI `/health`, pytest, ruff, GitHub Actions CI, branch protection.

**Docker primer** (start of Phase 1) — images vs containers, Dockerfile basics, Compose, volumes, ports. *(current phase)*

**Phase 1 — Naive RAG end to end.** Load docs → chunk (fixed size + overlap) → embed → store in pgvector → top-k similarity search → prompt with chunks → answer with citations. `/ingest` and `/query` endpoints. Postgres in Docker Compose.
- **Database migrations with Alembic** from the first table: create the `documents`/`chunks` schema and the pgvector extension via migrations, never by hand. Practise a second migration (e.g. add a metadata column) and a downgrade. Later, migrations run automatically as a deploy step in Phase 6.
- Checkpoint: Why do chunks overlap? What does cosine similarity measure? Why not put the whole document in the prompt? What if the right chunk isn't in top-k?

**Phase 2 — Evals (early, on purpose).** Golden dataset of 50–100 Q&A + source chunks (hand-written + LLM-generated, reviewed). Retrieval metrics: recall@k, MRR, hit rate. Generation metrics: faithfulness, answer relevance, correctness (LLM-as-judge, or Ragas/DeepEval). Evals run in CI; PR fails if quality drops below threshold.

**Phase 3 — Better retrieval, each change measured.** Chunking strategies (semantic, by heading), hybrid search (BM25 + vector), reranker (cross-encoder), query rewriting/HyDE, metadata filters. Keep a results table: change → metric delta → latency cost.

**Phase 4 — Dockerize properly + authentication.** Multi-stage Dockerfile, non-root user, health checks, Compose (api, db, worker), config via env vars. **API-key authentication** (FastAPI dependency, keys stored hashed in Postgres, 401 vs 403, never log keys) — the API must not be public before it goes to AWS in Phase 5.

**AWS primer** (before Phase 5) — account setup, IAM basics, **billing alarm**, regions, core services.

**Phase 5 — AWS via Terraform.** ECS Fargate (API + worker) behind ALB, RDS Postgres + pgvector, S3 for raw docs, SQS for async ingestion, Secrets Manager, CloudWatch. ⚠️ RDS + ALB ≈ $50+/month — `terraform destroy` when idle.

**Phase 6 — CI/CD.** PR: lint + tests + evals. Merge to main: build image → push to ECR → run DB migrations → deploy to staging → manual approval → prod. GitHub OIDC to AWS (no long-lived keys).

**Phase 7 — System design layer.** Exact + semantic caching (Redis/ElastiCache), streaming (SSE), rate limiting, timeouts/retries, tracing per stage (retrieve → rerank → generate) with latency + token cost, stateless scaling, HNSW tuning, guardrails (prompt injection, PII), feedback loop into eval set, multi-tenancy. Architecture doc with diagrams and trade-offs.
- **Authorization for multi-tenancy:** each API key belongs to a tenant; retrieval only returns that tenant's chunks (enforced in the query, not the prompt).
- **Load testing with Locust or k6:** baseline *before* Phase 7 optimisations, re-run after. Record p50/p95/p99 latency, throughput, error rate; find the bottleneck. Load-test staging, never prod; watch the AWS bill.

**Backlog (optional, later):** alerting on error-rate/latency spikes, DB backups & disaster recovery, blue/green or canary deploys with rollbacks, data/embedding versioning and re-embedding on model change.

## Open decision: frontend (revisit later)
(1) Streamlit — Python-only, fast; (2) plain HTML/JS or HTMX served by FastAPI — teaches CORS, streaming; (3) React/Next.js — production-grade but a second learning track. Suggested: Streamlit **debug UI** in Phase 1 (retrieved chunks, scores, exact prompt), real chat UI around Phase 7 (S3 + CloudFront) if wanted. Not decided.

## Phase 0 — detailed (current)
**Goal:** GitHub repo with a tiny API, a test, and CI checking every change.

**Target structure**
```
rag-project/
├── src/rag/main.py
├── tests/test_health.py
├── docs/learning-log.md
├── .github/workflows/ci.yml
├── pyproject.toml
├── uv.lock
├── CLAUDE.md
└── README.md
```

**Step-by-step guide (Parts A–K) and status**
- [x] **A. Tools** — Homebrew (added to PATH via `~/.zprofile`), uv, gh, git 2.56 (Homebrew). git user.name/email set.
- [x] **B. Project** — `uv init --package --name rag --python 3.12 rag-project`; `uv add fastapi uvicorn`; `uv add --dev pytest httpx ruff`. VS Code interpreter = `.venv/bin/python`. Branch renamed `master` → `main` (no commits yet).
- [x] **C. `/health`** — `src/rag/main.py`: FastAPI instance `app`; `GET /health` → `{"status": "ok"}`. Run `uv run uvicorn rag.main:app --reload`, explore `/docs`, `curl -i`. Break it: rename `app`.
- [x] **D. Test** — `tests/test_health.py` with `TestClient`: assert status 200 and body. `uv run pytest`. Break it: expect `"okay"`.
- [x] **E. Ruff** — `ruff check .`, `ruff format .`, `ruff format --check .`; try an unused import + bad spacing.
- [x] **F. GitHub** — first commit (confirm `.venv` not tracked, `uv.lock` is); `gh auth login`; `gh repo create rag-project --public --source=. --push` (public because free-plan private repos lack branch protection).
- [x] **G. CI** — `.github/workflows/ci.yml`: on push/PR to `main`; `ubuntu-latest`; `actions/checkout` + `astral-sh/setup-uv` (check latest major versions); `uv sync --locked`; ruff check; ruff format --check; pytest. Job name `checks`.
- [x] **H. Branch protection** — Settings → Rules → Rulesets on default branch: restrict deletions, block force pushes, require PR (0 approvals), require status check `checks`.
- [x] **I. Daily flow** — branch → commit → push → `gh pr create --fill` → `gh pr checks --watch` → `gh pr merge --squash --delete-branch` → pull main.
- [x] **J. Break it on purpose** — failing-test PR (merge blocked); bad-formatting PR (which step catches it?); thought experiment: no `uv.lock` + `--locked`.
- [x] **K. Learning log** — Phase 0 entry + checkpoint answers, merged via PR.

**Checkpoint questions (answer in learning log)**
1. Dependency vs dev dependency — why does it matter for a Docker image later?
2. Why does CI install from scratch instead of reusing my machine's setup?
3. `uses:` vs `run:` in a workflow step?
4. Why `ruff format --check` in CI instead of `ruff format`?
5. Why might a load balancer need `/health`?

**Next:** share `ci.yml` + checkpoint answers for review → Docker primer → Phase 1.

## Beginner gotchas already hit (don't re-explain from scratch, but watch for them)
- After installing Homebrew, its "Next steps" PATH lines must be run.
- `.venv/bin/activate` must be `source`d, not executed. With uv, activation is optional — prefer `uv run <cmd>`.
- `uv run` with no command just lists available commands.
- Run project commands from inside `rag-project/` (check the prompt / `pwd`); git only searches *up* for `.git`.
- Open `rag-project` (not the parent `Codelyf`) as the VS Code folder; Claude Code chats are stored per folder.

## Notes: why uv instead of pip
- pip only installs. uv covers install + venv + lockfile + Python versions + `uv run` in one tool.
- Key reason is the **lockfile**: `pip freeze` captures whatever's installed (incl. leftovers), mixes direct and transitive deps, and is platform-specific. uv: direct deps in `pyproject.toml`, full cross-platform resolution with hashes in `uv.lock`.
- Much faster (Rust, parallel, caching) → faster CI and Docker builds.
- `uv run` avoids "forgot to activate the venv" bugs.
- Trade-offs: pip is the universal default; uv is newer and company-backed (Astral). Both install the same packages from PyPI. `uv pip` mirrors pip.
- Optional exercise: build the same app with venv + pip + `pip freeze`, compare to `pyproject.toml` + `uv.lock`.
