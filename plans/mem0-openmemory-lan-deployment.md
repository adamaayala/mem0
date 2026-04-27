# OpenMemory LAN deployment (official repo) — implementation plan

> **For agentic workers:** Execute tasks in order. Each task has checkboxes; complete and verify before moving on. Use a dedicated git branch. Do not change unrelated packages (`mem0-ts/`, `server/`, CI workflows) unless a task explicitly says so.

**Goal:** Run OpenMemory from this monorepo on a LAN host with API on **8765**, UI on **3000**, **PostgreSQL + pgvector** as the vector store (no Qdrant container), **Ollama** reachable for LLM and embedder, compose suitable for production-shaped LAN use (healthchecks, no broken uvicorn flags, internal DB only).

**Architecture:** Single Docker Compose stack: `openmemory-mcp` (FastAPI + MCP), `openmemory-ui` (Next.js), `postgres` (pgvector image). The Mem0 `Memory` client reads `PG_*` env vars for pgvector (`openmemory/api/app/utils/memory.py`). OpenMemory app metadata uses SQLAlchemy `DATABASE_URL` (`openmemory/api/app/database.py`). Two logical databases on one Postgres instance: one for OpenMemory ORM, one for pgvector workload (avoids mixing Alembic-managed tables with mem0 collection tables in one DB unless you intentionally collapse them later).

**Tech stack:** Docker Compose, Python 3.12 (`openmemory/api/Dockerfile`), Next.js standalone (`openmemory/ui/Dockerfile`), PostgreSQL with pgvector, `mem0ai` Python package, optional Ollama on the LAN.

**Repo pin:** Before implementation, record here the exact **commit SHA** of `https://github.com/mem0ai/mem0` you are building from (agent: write the SHA into this section on first commit).

**Explicit non-goals (do not implement in this plan):**

- Pangolin, Traefik, Pocket ID, or public internet exposure (Phase 2 is separate).
- Migrating to `server/` + dashboard as the primary UI (different product: REST-first, **no MCP** in tree; would break “keep MCP on 8765” without new work).
- Adding first-class multi-user JWT login to OpenMemory UI (upstream OpenMemory is **not** the JWT wizard stack; LAN trust boundary is assumed unless you add a follow-up plan).

**Auth reality check:** OpenMemory today does **not** ship the same JWT + `/setup` flow as `server/`. Success criteria must **not** require “create account and log in” like the self-hosted server dashboard. Validation is: UI loads, `USER` / `NEXT_PUBLIC_USER_ID` align, MCP works with the chosen `client_name` and `user_id`.

---

## Preconditions (human or CI)

- [ ] Docker and Docker Compose v2 available on the target host.
- [ ] Operator provides: `OPENAI_API_KEY` **or** full Ollama settings (`LLM_PROVIDER`, `EMBEDDER_*`, `OLLAMA_BASE_URL` reachable from containers — often `http://192.168.x.x:11434` on Linux LAN, not `localhost`).
- [ ] Operator sets stable `USER` (string id for MCP path and UI); this is the OpenMemory user id, not a JWT subject from `server/`.

---

## Task 0: Branch and pin

**Files:** `plans/mem0-openmemory-lan-deployment.md` (this file)

- [ ] Create branch `deploy/openmemory-lan-pgvector` (or similar).
- [ ] Fill in **Repo pin** SHA at top of this doc after `git rev-parse HEAD`.

---

## Task 1: Fix SQLAlchemy engine for PostgreSQL

**Files:**

- Modify: `openmemory/api/app/database.py`

**Problem:** `connect_args={"check_same_thread": False}` is SQLite-specific. Passing it to PostgreSQL drivers can fail at runtime.

**Steps:**

- [ ] Build `connect_args` only when `DATABASE_URL` indicates SQLite (e.g. starts with `sqlite`).
- [ ] For non-SQLite URLs, omit `check_same_thread` (empty dict or omit `connect_args`).
- [ ] If tests exist for DB layer, extend or add a small unit test that `create_engine` receives correct `connect_args` for a fake `postgresql://` URL (optional but preferred).

**Verify:** Run OpenMemory API tests if present: `cd openmemory/api && pytest tests/ -q` (install deps per `requirements.txt` if needed).

---

## Task 2: Compose — Postgres + pgvector, remove Qdrant from default path

**Files:**

- Modify: `openmemory/docker-compose.yml`
- Create: `openmemory/init-db/` with an init SQL or shell script (see below)
- Modify: `openmemory/api/.env.example` (document `DATABASE_URL`, `PG_*`)

**Steps:**

- [ ] Add service `postgres`:
  - Image aligned with rest of monorepo (e.g. `ankane/pgvector` tag used in `server/docker-compose.yaml` or current supported pgvector image).
  - **No** `ports:` published to host (LAN DB internal to compose network only), unless operator override is explicitly documented as dev-only.
  - Volume for data persistence.
  - `healthcheck` using `pg_isready`.
- [ ] Mount `init-db` into `/docker-entrypoint-initdb.d` so two databases are created, e.g. `openmemory_app` and `mem0_vector` (names adjustable; document in `.env.example`).
- [ ] Remove service `mem0_store` (Qdrant) from this compose file **or** move Qdrant behind a Compose `profile: qdrant` so default `docker compose up` does not start it.
- [ ] Update `openmemory-mcp`:
  - `depends_on: postgres` with `condition: service_healthy`.
  - Environment (with defaults suitable for compose DNS name `postgres`):
    - `DATABASE_URL=postgresql+psycopg2://USER:PASS@postgres:5432/openmemory_app` (use real user/password from compose env; match init script).
    - `PG_HOST=postgres`, `PG_PORT=5432`, `PG_DB=mem0_vector`, `PG_USER=...`, `PG_PASSWORD=...` (same postgres instance, second database).
  - Remove `depends_on` / volume references to Qdrant when Qdrant is not in default profile.
- [ ] Fix API command: **do not** combine `--reload` with `--workers 4`. For LAN “prod-shaped” default use either:
  - `uvicorn main:app --host 0.0.0.0 --port 8765 --workers 4` (no reload), **or**
  - two compose overrides / profiles: `dev` (reload, 1 worker) vs `lan` (workers, no reload). Document which is default.

**Verify:**

- [ ] From `openmemory/`: `docker compose build && docker compose up -d`
- [ ] `docker compose ps` shows `postgres` and `openmemory-mcp` healthy (after Task 3 adds healthcheck for API if missing).
- [ ] `docker compose logs openmemory-mcp` shows pgvector auto-detection log line from `get_default_memory_config()` and no startup traceback.

---

## Task 3: Healthchecks and UI service

**Files:**

- Modify: `openmemory/docker-compose.yml`

**Steps:**

- [ ] Add `healthcheck` for `openmemory-mcp` hitting a lightweight HTTP endpoint (e.g. OpenAPI `GET /docs` returns 200, or add `GET /api/health` if you prefer a dedicated route — if you add a route, implement it in `openmemory/api/main.py` or a tiny router and test it).
- [ ] Add `healthcheck` for `openmemory-ui` (wget/curl to `http://localhost:3000` or existing health path used in `server/docker-compose.yaml` pattern).

**Verify:** `docker compose ps` — all services `healthy` (or document acceptable `running` if healthchecks deferred, but prefer healthy).

---

## Task 4: Environment documentation for Ollama on LAN

**Files:**

- Modify: `openmemory/api/.env.example`
- Optional: short `openmemory/DEPLOY-LAN.md` (only if `README.md` sunsetting banner should stay minimal)

**Steps:**

- [ ] Document that from **Linux** Docker on a LAN server, `OLLAMA_BASE_URL` must often be `http://<ollama-host-LAN-IP>:11434`, not `localhost`.
- [ ] Document `NEXT_PUBLIC_API_URL` for UI (e.g. `http://192.168.4.102:8765`) and `NEXT_PUBLIC_USER_ID` / `USER` must match the MCP URL segment.
- [ ] Document MCP URL pattern: `http://<api-host>:8765/mcp/<client_name>/sse/<user_id>` — **`client_name` must match** what Claude / plugin sends (e.g. `claude` or `openmemory`; align with `openmemory/ui/components/dashboard/Install.tsx` examples).

**Verify:** A new operator can copy `.env.example` to `.env` files without reading Python source.

---

## Task 5: CORS (optional hardening)

**Files:**

- Modify: `openmemory/api/main.py` (CORSMiddleware)
- Optional: read allowed origins from env var e.g. `CORS_ALLOW_ORIGINS` (comma-separated), default `*` for backward compatibility.

**Steps:**

- [ ] If env set, split and pass explicit list; else keep `*`.

**Verify:** UI still loads from `http://<LAN-IP>:3000` when origin list includes that URL.

---

## Task 6: Final acceptance (operator)

Run on the target host with real Ollama / keys filled in.

- [ ] `docker compose ps` — postgres, API, UI up; API and UI healthy.
- [ ] UI loads at `http://<LAN-IP>:3000`.
- [ ] Create or view a memory from UI (no login wall — expected for OpenMemory).
- [ ] MCP: connect Claude Code (or test client) to `http://<LAN-IP>:8765/mcp/<client_name>/sse/<USER>`; add memory via tool; appears in UI.
- [ ] Search via MCP returns expected hit.
- [ ] `docker compose restart` — memories still present (Postgres volume persisted).

---

## Follow-up (separate plan, not in scope here)

- Internet Phase 2: reverse proxy, TLS, forward-auth, and syncing `NEXT_PUBLIC_API_URL` with public origin.
- If product direction shifts to **`server/`** only: new plan for REST + API keys + dashboard and **different MCP integration** (hosted MCP or custom bridge).

---

## Appendix: MCP and `.mcp.json`

Example shape (operator edits host and ids):

```json
{
  "mcpServers": {
    "mem0": {
      "url": "http://192.168.4.102:8765/mcp/claude/sse/your-user-id-from-USER-env"
    }
  }
}
```

Replace `claude` with the `client_name` you standardize on; replace `your-user-id-from-USER-env` with the same value as `USER` / dashboard user id.
