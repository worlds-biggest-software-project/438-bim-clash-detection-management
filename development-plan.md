# BIM Clash Detection & Management — Phased Development Plan

> Project: 438-bim-clash-detection-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. The database design is based on **data-model-suggestion-1** (Normalized Relational PostgreSQL + PostGIS), with one pragmatic concession borrowed from suggestion-3: IFC property sets are stored as `JSONB` rather than EAV rows to avoid the property-explosion performance problem noted in suggestion-1's cons. Event sourcing (suggestion-2) and graph storage (suggestion-4) are deliberately rejected for the MVP as over-engineered for an open-source tool seeking contributors.

---

## Product Summary

An AI-native, open-source platform that federates discipline-specific BIM models (architectural, structural, MEP) via the open IFC standard, runs configurable hard/soft clash detection, and manages the full issue-resolution lifecycle with BCF-standard interoperability. Differentiators versus Navisworks/ACC/Revizto: open-source and vendor-neutral (no NWD/NWC dependency), browser-based stakeholder access with no per-seat licence, BCF-API conformance for cross-tool issue exchange, and AI-assisted clash relevance scoring / natural-language querying to cut clash noise.

**Primary users**: BIM coordinators, design leads, MEP contractors, project managers. **Lightweight viewers**: subcontractors, clients, field teams.

**Deployment model**: Hybrid — Docker-Compose self-hosted (data-residency firms) and the same containers deployable to a managed cloud. API-first; browser SPA viewer; no desktop install required for any user.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The only mature open-source IFC geometry toolkit (`IfcOpenShell`, incl. `ifcopenshell.geom` and `bcf` modules) is Python/C++. Clash detection is geometry-heavy; building on IfcOpenShell avoids reimplementing IFC parsing. ML/AI scoring libraries (scikit-learn, sentence-transformers) are first-class in Python. |
| API framework | **FastAPI** | Async (needed for long IFC/clash jobs offloaded to a queue), auto-generates OpenAPI 3.1 (required by standards.md for client SDKs), Pydantic v2 validation maps cleanly to BCF JSON Schema payloads. |
| Database | **PostgreSQL 16 + PostGIS 3.4** | Per data-model-suggestion-1. Native 3D geometry (`ST_3DIntersects`, `ST_3DDWithin`) provides the broad-phase clash filter; JSONB handles open IFC property sets; mature migration/backup tooling suits self-hosting. |
| Spatial broad-phase | **PostGIS GiST index on AABB `PolyhedralSurfaceZ`** | Reduces N×M element pairs to candidate pairs before exact mesh testing. |
| Exact (narrow-phase) clash | **IfcOpenShell geometry + `fcl` (Flexible Collision Library) via `python-fcl`** | `fcl` gives mesh-mesh intersection and signed distance (for soft/clearance clashes) on the candidate pairs surviving broad-phase. |
| Object storage | **MinIO (S3-compatible)** | Self-hostable, stores raw IFC files, tessellated GLB geometry, and BCF snapshot PNGs out of the DB. Cloud deployments swap the endpoint for AWS S3 with no code change. |
| Task queue | **Celery + Redis** | IFC parsing and clash runs are minutes-long CPU-bound jobs that must not block the API. Redis doubles as result/dashboard cache. |
| Frontend | **React 18 + TypeScript + Vite** | SPA for the dashboard and issue management. |
| 3D viewer | **`@thatopen/components` (formerly openBIM-components) over Three.js** | Open-source, IFC/fragments-native browser viewer with element selection and section planes — provides BCF viewpoint camera + component highlighting without a proprietary viewer (avoids APS Forge lock-in). |
| Auth | **OAuth 2.0 + OIDC (Authlib), JWT (RFC 7519) bearer tokens** | Required by standards.md; OIDC enables enterprise SSO. Local password auth (argon2) for self-hosted small teams. |
| BCF | **`ifcopenshell.bcf` for files; custom FastAPI router for BCF REST API v3.0** | standards.md flags BCF API v3.0 as THE key interop standard. |
| ML / AI | **scikit-learn (relevance/severity classifiers), sentence-transformers + pgvector (NL query & clustering)** | Lightweight, CPU-runnable, no external LLM dependency required for MVP scoring; NL query uses embedding+rules, optional LLM provider behind an interface. |
| Reporting | **WeasyPrint (PDF), Python `csv`** | Pure-Python HTML→PDF, no headless browser dependency. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Type-safe async access; versioned migrations for self-hosters. PostGIS via GeoAlchemy2. |
| Testing | **pytest, pytest-asyncio, testcontainers-python, Playwright** | Real Postgres/MinIO via containers for integration; Playwright for SPA e2e. |
| Code quality | **ruff (lint+format), mypy (strict), pre-commit** | Single fast linter/formatter; mypy enforces typed domain models. |
| Frontend tooling | **ESLint, Prettier, Vitest, TanStack Query** | Standard React stack. |
| Package mgmt | **uv (Python), pnpm (frontend)** | Fast, lockfile-based, reproducible. |
| Containerisation | **Docker + docker-compose** | One-command self-host: api, worker, postgres+postgis, redis, minio, frontend. |
| CI | **GitHub Actions** | Lint, type-check, test matrix, Docker build. |

### Project Structure

```
bim-clash/
├── README.md
├── LICENSE                        # AGPL-3.0 (open-source, copyleft for SaaS competitors)
├── docker-compose.yml
├── docker-compose.dev.yml
├── .github/workflows/ci.yml
├── backend/
│   ├── pyproject.toml             # uv-managed
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── migrations/                # Alembic revisions
│   ├── app/
│   │   ├── main.py                # FastAPI app factory, router mounting
│   │   ├── config.py              # Pydantic Settings (env-driven)
│   │   ├── db.py                  # async engine, session, PostGIS init
│   │   ├── auth/                  # OAuth/OIDC/JWT, password hashing, RBAC deps
│   │   ├── models/                # SQLAlchemy ORM models (tables)
│   │   ├── schemas/               # Pydantic request/response + BCF schemas
│   │   ├── api/
│   │   │   ├── projects.py
│   │   │   ├── models_router.py   # model file upload/version
│   │   │   ├── federation.py
│   │   │   ├── clash_tests.py
│   │   │   ├── clashes.py
│   │   │   ├── issues.py
│   │   │   ├── bcf/               # BCF REST API v3.0 router (versions, projects, topics, comments, viewpoints)
│   │   │   ├── reports.py
│   │   │   ├── analytics.py
│   │   │   └── nl_query.py
│   │   ├── services/              # business logic (no FastAPI imports)
│   │   │   ├── ifc_parser.py
│   │   │   ├── geometry.py        # tessellation, AABB, GLB export
│   │   │   ├── clash_engine.py    # broad + narrow phase
│   │   │   ├── clash_tracking.py  # stable hashing, run diffing
│   │   │   ├── bcf_io.py          # BCF file import/export
│   │   │   ├── reporting.py
│   │   │   └── storage.py         # MinIO abstraction
│   │   ├── ml/
│   │   │   ├── features.py        # clash → feature vector
│   │   │   ├── relevance.py       # classifier train/predict
│   │   │   ├── clustering.py
│   │   │   └── nl_parser.py       # NL → structured filter
│   │   ├── tasks/                 # Celery tasks
│   │   │   ├── celery_app.py
│   │   │   ├── parse_model.py
│   │   │   └── run_clash_test.py
│   │   └── integrations/
│   │       └── webhooks.py
│   └── tests/
│       ├── conftest.py            # fixtures: db container, minio, sample IFC
│       ├── fixtures/              # sample .ifc files, expected clash JSON
│       ├── unit/
│       ├── integration/
│       └── e2e/
└── frontend/
    ├── package.json               # pnpm
    ├── Dockerfile
    ├── vite.config.ts
    ├── src/
    │   ├── api/                   # generated OpenAPI client + TanStack hooks
    │   ├── components/
    │   ├── viewer/                # @thatopen/components 3D viewer wrapper
    │   ├── pages/                 # Projects, Models, ClashMatrix, ClashList, IssueBoard, Dashboard
    │   └── auth/
    └── tests/
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the runnable skeleton: containerised Postgres+PostGIS, FastAPI app, config, migrations, auth primitives, and CI. After this phase a developer can `docker compose up`, hit a health endpoint, register/log in, and run the test suite. Everything later builds on these abstractions.

### Tasks

#### 1.1 — Repository, tooling, and Docker Compose
**What**: Scaffold the monorepo, dependency management, linting/type-checking, and a one-command dev environment.

**Design**:
- `backend/pyproject.toml` with deps: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `geoalchemy2`, `asyncpg`, `alembic`, `pydantic-settings`, `authlib`, `argon2-cffi`, `python-jose`, `celery[redis]`, `redis`, `minio`, `ifcopenshell`, `python-fcl`, `numpy`, `weasyprint`; dev: `pytest`, `pytest-asyncio`, `testcontainers`, `ruff`, `mypy`, `pre-commit`.
- `docker-compose.yml` services: `postgres` (image `postgis/postgis:16-3.4`), `redis`, `minio`, `api`, `worker`, `frontend`. Named volumes for pg and minio data.
- `app/config.py`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://redis:6379/0"
    minio_endpoint: str
    minio_access_key: str
    minio_secret_key: str
    minio_bucket: str = "bim-clash"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_expiry_seconds: int = 3600
    oidc_issuer: str | None = None
    model_config = SettingsConfigDict(env_prefix="BIMCLASH_")
```
- `.github/workflows/ci.yml`: jobs `lint` (ruff check + format --check), `typecheck` (mypy app), `test` (pytest with services), `build` (docker build api + frontend).

**Testing**:
- `Unit: Settings loads from env vars with BIMCLASH_ prefix → correct typed fields, defaults applied`
- `Unit: missing required env (jwt_secret) → ValidationError naming the field`
- `Integration: docker compose config validates (compose config exits 0)`

#### 1.2 — Database engine, session, PostGIS bootstrap
**What**: Async SQLAlchemy engine, session dependency, and PostGIS extension enablement.

**Design**:
- `app/db.py`: `create_async_engine(settings.database_url)`, `async_sessionmaker`, FastAPI dependency `get_session()` yielding a session with commit/rollback.
- Alembic initial migration runs `CREATE EXTENSION IF NOT EXISTS postgis;` before any spatial table.
- Base declarative class with `id UUID default gen_random_uuid()`, `created_at`, `updated_at` mixin.

**Testing**:
- `Integration (testcontainers postgis): engine connects, SELECT PostGIS_Version() returns a version string`
- `Integration: get_session commits on success, rolls back on raised exception`

#### 1.3 — Identity & multi-tenancy schema + auth
**What**: `organizations`, `users`, `organization_memberships`, `projects`, `project_memberships`, `disciplines` tables (DDL from data-model-suggestion-1) plus registration/login and RBAC dependencies.

**Design**:
- ORM models mirroring suggestion-1 DDL for the six tables above.
- Password auth: argon2 hash in `users.password_hash`; OIDC path sets `auth_provider='oidc'`.
- `POST /auth/register` (org owner bootstrap), `POST /auth/login` → `{access_token, token_type:"bearer"}` JWT (RFC 7519) with `sub=user_id`, `org` claim.
- RBAC dependency factory:
```python
def require_project_role(*roles: str):
    async def dep(project_id: UUID, user=Depends(current_user), s=Depends(get_session)) -> ProjectMembership:
        m = await get_membership(s, project_id, user.id)
        if m is None or m.role not in roles:
            raise HTTPException(403)  # OWASP API1: object-level authz
        return m
    return dep
```
- Project roles: `coordinator, lead, member, viewer`. Org roles: `owner, admin, member, viewer`.

**Testing**:
- `Unit: argon2 verify round-trips; wrong password → False`
- `Integration: register → login → JWT decodes with correct sub and org claims`
- `Integration: GET /projects/{id} as non-member → 403 (broken-object-level-authz guard, OWASP API1)`
- `Integration: expired JWT → 401`

### Definition of Done
All of: tables migrate cleanly, auth flow works, RBAC enforced, ruff+mypy pass, CI green, `docker compose up` serves `GET /health` 200.

---

## Phase 2: Model Ingestion & IFC Parsing

### Purpose
Turn uploaded IFC files into queryable, geometry-bearing BIM elements in the database. This is the data foundation the clash engine consumes. After this phase, a coordinator can upload an IFC, watch it process asynchronously, and see parsed elements with bounding boxes, storeys, classifications, and property sets.

### Tasks

#### 2.1 — Model file & version upload with MinIO storage
**What**: Upload endpoints that store raw IFC in MinIO and create `model_files` / `model_versions` rows; re-upload creates a new version.

**Design**:
- Tables `model_files`, `model_versions`, `federated_models`, `federated_model_members` from suggestion-1.
- `app/services/storage.py`: `put_object(key, stream) -> path`, `presigned_get(path) -> url`, `get_object(path) -> bytes`.
- `POST /projects/{id}/models` (multipart): validates `source_format ∈ {IFC2X3, IFC4, IFC4X3_ADD2}` by sniffing the IFC `FILE_SCHEMA` header; computes `checksum_sha256`; stores to `projects/{pid}/models/{mid}/v{n}.ifc`; sets `upload_status='processing'`; enqueues `parse_model` Celery task; returns `202` with model_version id.
- Version number auto-increments per `model_file_id`.

**Testing**:
- `Integration: upload valid IFC4 → 202, model_version row with checksum, object in MinIO`
- `Integration: re-upload same model_file → version_number increments to 2`
- `Unit: schema sniff on IFC2X3 header → 'IFC2X3'; on non-IFC text → 400`
- `Integration: upload as project 'viewer' role → 403`

#### 2.2 — IFC parser service (Celery task)
**What**: Background task using IfcOpenShell to extract elements, spatial containment, classifications, properties, and bounding boxes into the DB.

**Design**:
- `bim_elements` table per suggestion-1, but `bim_element_properties` replaced by a `properties JSONB` column on `bim_elements` (suggestion-3 concession) shaped `{psetName: {propName: {value, type, unit}}}`. Keep a GIN index on `properties`.
- `app/services/ifc_parser.py`:
```python
@dataclass
class ParsedElement:
    ifc_global_id: str
    ifc_entity_type: str          # IfcWall, IfcDuctSegment, ...
    ifc_name: str | None
    building_name: str | None
    storey_name: str | None
    classification_system: str | None
    classification_code: str | None
    material_name: str | None
    bbox_min: tuple[float, float, float]
    bbox_max: tuple[float, float, float]
    centroid: tuple[float, float, float]
    volume_m3: float | None
    properties: dict[str, dict]
```
- Logic: open file with `ifcopenshell.open`; iterate `IfcElement` subtypes; resolve storey via `IfcRelContainedInSpatialStructure`; classification via `IfcRelAssociatesClassification`; material via `IfcRelAssociatesMaterial`; psets via `ifcopenshell.util.element.get_psets`. Geometry via `ifcopenshell.geom` with world coords → compute AABB and centroid in NumPy; write PostGIS `bbox_geom` (PolyhedralSurfaceZ box) and `centroid` (PointZ).
- Task updates `model_versions.processing_status` (pending→running→completed/failed), writes `element_count`, `processing_log`. On failure stores `processing_error`.
- Coordinate units normalised to metres using the IFC `IfcUnitAssignment`.

**Testing**:
- `Fixture: tests/fixtures/duplex.ifc (public buildingSMART sample) parses → known element_count within tolerance`
- `Unit: storey resolution for an element in a known storey → correct storey_name`
- `Unit: unit conversion mm-model → bbox values in metres`
- `Integration: malformed IFC → task sets processing_status='failed', processing_error populated, no partial elements left (transaction rollback)`
- `Unit: get_psets output maps into properties JSONB with value/type/unit`

#### 2.3 — Tessellated geometry export for the viewer
**What**: Produce a GLB mesh per model version for the browser viewer, stored in MinIO.

**Design**:
- During parse, tessellate geometry and pack into a glTF/GLB keyed by IFC GlobalId in `mesh.extras` (so the viewer can map mesh nodes ↔ elements/clashes). Store at `projects/{pid}/models/{mid}/v{n}.glb`; record path on `model_versions`.
- `GET /models/{version_id}/geometry` → presigned MinIO URL.

**Testing**:
- `Integration: parsed model produces GLB; downloaded GLB is valid glTF (parses with pygltflib), node count == element_count`
- `Unit: each GLB node extras carries the source ifc_global_id`

### Definition of Done
Upload→parse→elements+GLB pipeline works end-to-end on the sample IFC; processing states transition correctly; failures are isolated; spatial indexes exist; tests pass; migrations created.

---

## Phase 3: Clash Detection Engine (Core Value)

### Purpose
The heart of the product. Configure discipline-pair clash tests with tolerances and run hard/soft detection over federated models, producing stable, trackable clash results. After this phase the platform delivers its primary value: federate models → detect clashes → list results.

### Tasks

#### 3.1 — Clash test configuration
**What**: CRUD for `clash_tests` (and `clash_test_templates`, `clash_ignore_rules`) per suggestion-1.

**Design**:
- `POST /projects/{id}/clash-tests` body (Pydantic):
```python
class ClashTestCreate(BaseModel):
    name: str
    federated_model_id: UUID
    test_type: Literal["hard", "soft", "clearance"]
    set_a: SelectionSet   # discipline_id, model_version_id, ifc_types[], storey_filter[], search_query?
    set_b: SelectionSet
    tolerance_mm: float = 0.0
    clearance_mm: float | None = None
    exclude_same_discipline: bool = False
```
- Templates: save a test config as reusable (`is_global` org-wide or project-scoped) — addresses the ACC-only "reusable configuration" gap.
- Ignore rules: `ifc_type_pair`, `element_pair`, `property_match`, `spatial_zone`.

**Testing**:
- `Unit: clearance test without clearance_mm → 422 validation error`
- `Integration: create test from template copies selection sets and tolerances`
- `Integration: create test referencing federated_model from another project → 403`

#### 3.2 — Broad-phase candidate generation (PostGIS)
**What**: Reduce N×M element pairs to spatially-overlapping candidate pairs via PostGIS.

**Design**:
- Resolve set A and set B element id lists from selection filters (discipline, model_version, ifc_types, storey_filter, search_query against `properties` JSONB / `ifc_name`).
- For hard clashes: `ST_3DIntersects(a.bbox_geom, b.bbox_geom)`. For soft/clearance: `ST_3DDWithin(a.bbox_geom, b.bbox_geom, clearance_m)`.
- Apply `exclude_same_discipline` and ignore-rule filters in SQL. Chunk set A in batches (e.g. 5k) to bound memory on 500k-element models (addresses suggestion-1 scaling con).

**Testing**:
- `Integration: two overlapping boxes → 1 candidate pair; two distant boxes → 0`
- `Integration: clearance 50mm, gap 30mm → candidate; gap 80mm → none`
- `Unit: ignore rule ifc_type_pair(IfcDuctSegment,IfcCableCarrierSegment) removes those candidates`

#### 3.3 — Narrow-phase exact intersection
**What**: Exact mesh test on candidate pairs to confirm clashes and compute clash geometry.

**Design**:
- `app/services/clash_engine.py`: load each candidate element's tessellated mesh; build `fcl.BVHModel`; for hard → `fcl.collide` (contact point = `clash_point`, penetration depth → negative `clash_distance_mm`, intersection volume estimate → `intersection_volume_mm3`); for soft/clearance → `fcl.distance`, flag clash when `distance < clearance_mm` (`clash_distance_mm` positive).
- Returns:
```python
@dataclass
class ClashResult:
    element_a_id: UUID; element_b_id: UUID
    clash_point: tuple[float, float, float]
    clash_distance_mm: float        # negative=penetration, positive=clearance violation
    intersection_volume_mm3: float | None
    storey_name: str | None; grid_reference: str | None
```

**Testing**:
- `Fixture: pipe-through-beam pair → hard clash, negative distance, clash_point inside both AABBs`
- `Fixture: pipe 40mm from beam, clearance 50mm → soft clash distance≈40`
- `Unit: touching-but-not-overlapping (distance 0, hard) → boundary handling per tolerance_mm`
- `Performance: 10k candidate pairs complete < 60s (marked slow)`

#### 3.4 — Test runs, stable clash hashing, and cross-run diffing
**What**: Persist runs and clashes; assign stable hashes so a clash tracks across model versions; diff against the previous run.

**Design**:
- `clash_test_runs` and `clashes` tables per suggestion-1 (`clash_hash = SHA256(element_a_global_id + element_b_global_id + clash_test_id)`).
- `app/services/clash_tracking.py`: after a run, compute set difference vs previous run by `clash_hash` → populate `clashes_appeared`, `clashes_disappeared`, `clashes_unchanged`; carry forward lifecycle fields (`status`, `assigned_to`, `priority`) for unchanged clashes so coordination work is not lost on re-detection (the ACC/BIMcollab "Smart Issues auto-status" feature).
- `run_clash_test` Celery task orchestrates 3.2→3.3→persist→diff; updates run + test summary counts.
- `POST /clash-tests/{id}/runs` → 202; `GET /clash-tests/{id}/runs/{run}` returns summary.

**Testing**:
- `Unit: clash_hash deterministic and order-independent of pair listing (canonicalise A<B)`
- `Integration: run twice unchanged → all clashes 'unchanged', statuses preserved`
- `Integration: move an element out of collision, re-run → that clash 'disappeared', new ones 'appeared'`
- `Integration: resolved clash that persists keeps status='resolved' on next run`

#### 3.5 — Clash list & grouping API
**What**: Query, filter, sort, and group clashes; `clash_groups` / `clash_group_members`.

**Design**:
- `GET /clash-tests/{id}/clashes?status=&priority=&storey=&discipline_pair=&sort=ai_relevance&page=` — paginated (RFC 8288 `Link` headers).
- `POST /clash-tests/{id}/groups` group by `zone|level|discipline_pair` (auto) or manual selection.

**Testing**:
- `Integration: filter status=active returns only active; pagination Link header present`
- `Integration: auto-group by level → one group per distinct storey, correct counts`

### Definition of Done
Federate → configure test → run → results pipeline works on fixtures; clashes track stably across runs; lifecycle preserved on re-detection; grouping works; spatial + clash indexes present; tests (incl. one slow perf test) pass.

---

## Phase 4: Issue Lifecycle & BCF Interoperability

### Purpose
Convert clashes into tracked, assignable issues and make them portable to/from every other BIM tool via the open BCF standard — the vendor-neutral interoperability that is the project's core differentiator. After this phase, coordinators manage resolution and exchange issues with Revit/Navisworks/Solibri/BIMcollab.

### Tasks

#### 4.1 — Issue management
**What**: `issues`, `issue_comments`, `issue_viewpoints`, `issue_attachments` tables; clash→issue conversion; lifecycle.

**Design**:
- Tables per suggestion-1 (BCF-aligned field names). Lifecycle statuses: `Open, In Progress, Resolved, Closed` (BCF `TopicStatus`); also expose the clash statuses `New, Active, Reviewed, Approved, Resolved` from features.md on the clash itself.
- `POST /clashes/{id}/issue` creates an issue from a clash, copying element refs into a default viewpoint (`selected_components = [global_id_a, global_id_b]`) and the clash camera framing.
- `PATCH /issues/{id}` for status/assignee/priority/due_date; appends to `audit_log`.

**Testing**:
- `Integration: clash→issue creates issue + viewpoint with both component GUIDs selected`
- `Integration: status transition Open→Resolved writes audit_log old/new values`
- `Integration: assign to non-project-member → 422`

#### 4.2 — BCF file import/export (2.1 + 3.0)
**What**: Read/write `.bcfzip` archives via IfcOpenShell's bcf module.

**Design**:
- `app/services/bcf_io.py`: export selected issues → bcfzip (markup.bcf XML, viewpoint.bcfv, snapshot.png per topic); import maps Topics→issues, Comments→issue_comments, Viewpoints→issue_viewpoints (camera vectors, selected/exception components, clipping planes, coloring JSONB). Log to `bcf_exchanges`.
- `POST /projects/{id}/bcf/import` (multipart), `GET /projects/{id}/bcf/export?issue_ids=`.

**Testing**:
- `Fixture: import a sample bcfzip → issues+viewpoints+comments created, counts match markup`
- `Integration: export then re-import round-trips topic GUID, status, viewpoint camera (within float tolerance)`
- `Unit: snapshot PNG embedded and re-extracted byte-identical`

#### 4.3 — BCF REST API v3.0 endpoints
**What**: Conformant BCF-API so external BCF Manager plugins sync live without files.

**Design**:
- Router `app/api/bcf/` implementing v3.0: `GET /bcf/3.0/projects`, `/projects/{pid}/topics` (GET/POST/PUT), `/topics/{tid}/comments`, `/topics/{tid}/viewpoints`, `/projects/{pid}/extensions`. JSON bodies validated against BCF JSON Schema (Draft 2020-12). OAuth 2.0 bearer auth (RFC 6749). Our internal `issues` map to BCF Topics; `disciplines`/labels map to extensions.
- Pagination via RFC 8288 + `$top`/`$skip` query params per spec.

**Testing**:
- `Integration: GET /bcf/3.0/projects returns auth'd user's projects in BCF shape`
- `Integration: POST topic → issue created, GET topic returns matching JSON Schema-valid payload`
- `Integration: request without bearer token → 401`
- `Contract: responses validate against BCF v3.0 JSON Schema fixtures`

### Definition of Done
Clash→issue→resolve lifecycle works; BCF files round-trip with a real bcfzip fixture; BCF REST endpoints pass JSON-Schema contract tests and auth checks; audit trail recorded.

---

## Phase 5: Browser Viewer & Frontend (parallelisable with Phase 4)

### Purpose
Deliver the no-install browser experience that locks non-BIM stakeholders out of incumbents. After this phase, any permitted user views the federated 3D model, sees clashes highlighted in place, navigates the clash list, and manages issues on a Kanban board.

### Tasks

#### 5.1 — App shell, auth, project/model UI
**What**: React+Vite SPA with login, project list, model upload UI, generated typed API client.

**Design**:
- Generate the TS client from the FastAPI OpenAPI 3.1 schema (`openapi-typescript`); TanStack Query hooks. Auth stores JWT; attaches bearer header; refresh on 401.
- Pages: Login, Projects, Project→Models (upload + processing status polling), Federated Model builder.

**Testing**:
- `Vitest: auth store persists token, clears on logout`
- `Playwright e2e: log in → create project → upload IFC → see status reach 'ready'`

#### 5.2 — 3D viewer with clash highlighting
**What**: Embed `@thatopen/components` viewer loading the model GLB(s), highlight clashing elements, navigate to a clash.

**Design**:
- `viewer/` wrapper loads GLBs for federated model members; maps `ifc_global_id` (from GLB node extras) → mesh fragments. Selecting a clash colours element A/B (red/orange), frames the camera to the clash point, shows a section plane option. Camera state serialises to a BCF viewpoint for issue creation.

**Testing**:
- `Playwright e2e: open clash test → click a clash row → viewer highlights two elements and recenters`
- `Unit: camera state → BCF viewpoint vectors conversion matches backend expectation`

#### 5.3 — Clash matrix, clash list, issue Kanban, reports UI
**What**: Coordinator workflow screens.

**Design**:
- Clash matrix grid (discipline pairs × counts, colour-coded status) → drill into clash list (filter/sort/group, bulk status/assign). Issue Kanban (Open/In Progress/Resolved/Closed) with drag-to-transition. Report download buttons (PDF/CSV) and BCF import/export controls.

**Testing**:
- `Playwright e2e: filter clashes by storey, bulk-assign to a user, verify backend reflects change`
- `Playwright e2e: drag issue card Open→Resolved → status persists on reload`

### Definition of Done
A stakeholder with only a browser can view the federated model, see highlighted clashes, manage issues, and export reports/BCF; e2e flows green; frontend lint/type/test pass; frontend Docker image builds.

---

## Phase 6: Reporting, Analytics & Automated Re-detection

### Purpose
Deliver the contractual deliverables (clash reports) and the management visibility (dashboards) incumbents charge premium tiers for, plus auto re-detection on upload. After this phase the platform supports coordination governance, not just detection.

### Tasks

#### 6.1 — Clash reports (PDF/CSV)
**What**: Generate report deliverables.

**Design**:
- `app/services/reporting.py`: CSV (one row per clash with element refs, status, location, assignee, AI scores); PDF via WeasyPrint from an HTML template (summary stats, discipline matrix, per-clash cards with snapshot). `report_templates` table stores saved filter+format configs. `GET /clash-tests/{id}/report?format=pdf|csv`.

**Testing**:
- `Integration: CSV report row count == clash count; columns present`
- `Integration: PDF generates non-empty valid PDF (magic bytes), contains test name`

#### 6.2 — Dashboard analytics & trend snapshots
**What**: Open/closed trends, team workload, bottleneck detection.

**Design**:
- `clash_daily_snapshots` table; a nightly Celery beat task writes per-test daily counts. `GET /projects/{id}/analytics/trends`, `/workload`. Redis-cache aggregations.

**Testing**:
- `Integration: snapshot task writes one row per active test per day, counts match live data`
- `Integration: workload endpoint aggregates assigned/resolved per user correctly`

#### 6.3 — Automated re-detection on model upload
**What**: Re-run affected clash tests automatically when a new model version is parsed, with change-diff highlighting.

**Design**:
- On `ModelVersionProcessingCompleted`, find clash tests whose selection sets reference that model file and enqueue runs; the run diff (Phase 3.4) surfaces appeared/disappeared/unchanged. Optional webhook fires `clash.run.completed` (`app/integrations/webhooks.py`).

**Testing**:
- `Integration: upload new version of a model referenced by a test → run auto-enqueued and completes`
- `Integration: webhook POST fired with HMAC signature on run completion`

### Definition of Done
Reports export correctly; dashboards reflect live data; new uploads trigger re-detection with correct diffs; scheduled snapshots run; tests pass.

---

## Phase 7: AI-Augmented Coordination

### Purpose
Realise the AI-native advantage: cut clash noise, prioritise rework risk, cluster root causes, and let coordinators query in natural language. This is the strongest differentiator versus all incumbents (none offer built-in AI filtering).

### Tasks

#### 7.1 — Feature extraction & relevance/severity scoring
**What**: ML classifiers scoring each clash for true-positive relevance and rework-cost severity.

**Design**:
- `ml_training_samples`, `ml_models` tables per suggestion-1. `ml/features.py` builds a feature vector per clash: ifc-type pair (one-hot), discipline pair, penetration depth, intersection volume, element sizes, storey, count of similar clashes, whether elements share a host. `ml/relevance.py`: scikit-learn `GradientBoostingClassifier`; trains on resolved clashes labelled `false_positive` (irrelevant) vs real; writes `ai_relevance_score`, `ai_severity_score` to `clashes`. Cold-start: heuristic scorer (tiny penetration + small elements + insulation/cable-tray pairs → low relevance) until enough labels.
- Scoring runs as a post-step in `run_clash_test`. `ml_models.is_active` selects the serving model.

**Testing**:
- `Unit: feature vector deterministic for a fixed clash`
- `Unit: heuristic scorer ranks a 1mm cable-tray graze below a 200mm pipe-beam penetration`
- `Integration: after a run, clashes have relevance scores; list sortable by ai_relevance`
- `Integration: train on labelled fixtures → metrics (auc) recorded in ml_models`

#### 7.2 — Root-cause clustering
**What**: Group related clashes by shared cause (e.g. one misaligned grid).

**Design**:
- `ml/clustering.py`: DBSCAN over (clash_point xyz + discipline pair + ifc-type pair embedding); cluster id → `clashes.ai_cluster_id`, materialised as `clash_groups(group_type='ai_cluster')`.

**Testing**:
- `Fixture: 20 clashes along one misaligned wall cluster into 1 group; scattered clashes stay separate`

#### 7.3 — Natural-language query interface
**What**: "show me all MEP-structural clashes above Level 3" → structured filter.

**Design**:
- `ml/nl_parser.py`: parse NL → `ClashFilter` (disciplines, storeys with comparators, status, priority, ifc types). MVP: rule/grammar + embedding match (sentence-transformers + pgvector) over a synonym lexicon (discipline names, storey patterns). Behind `NLParser` interface so an optional LLM provider can be plugged. Log to `nl_query_log` with `parsed_filters` and feedback. `POST /projects/{id}/clashes/nl-query`.
- Optional MCP server (standards.md) exposing clash query + issue update as tools, deferred to backlog.

**Testing**:
- `Unit: "MEP-structural clashes above Level 3" → disciplines={MEP,Structural}, storey>Level 3`
- `Unit: "unresolved critical clashes assigned to Sara" → status≠resolved, priority=critical, assignee=Sara`
- `Integration: nl-query endpoint returns same clashes as the equivalent structured filter`

### Definition of Done
Clashes are scored and sortable by relevance/severity; clusters surface in the UI; NL queries return correct results and are logged; heuristic cold-start works without training data; tests pass.

---

## Phase 8: Integrations, Hardening & Release

### Purpose
Make the platform production-deployable and connected: external coordination platforms, security hardening, performance at scale, and release packaging.

### Tasks

#### 8.1 — External platform integrations
**What**: BCF-based sync with Autodesk ACC, BIMcollab, Procore (where BCF/REST available).

**Design**:
- `integrations/`: connector interface `push_issues()/pull_issues()` implemented over the partner's BCF API or REST; OAuth 2.0 credential storage per project (encrypted). ACC via APS Model Coordination read; BIMcollab/Newforma via BCF API; Procore via REST issue mapping.

**Testing**:
- `Integration (mocked): pull from a mock BCF-API server creates local issues; push sends conformant topics`
- `Unit: OAuth token refresh on 401 from partner`

#### 8.2 — Security hardening (OWASP API Top 10)
**What**: Systematic authz, rate limiting, input validation, TLS, secrets.

**Design**:
- Object-level authz checks on every project-scoped resource (API1); response schemas whitelist fields to prevent excessive data exposure (API3); UUID ids (no IDOR, API1); rate limiting (Redis) on auth + upload; TLS 1.3 termination documented for prod; secrets via env only. Add an automated test sweep asserting cross-tenant access is blocked on all routers.

**Testing**:
- `Integration: cross-org access attempt on every resource type → 403 (parametrised sweep)`
- `Integration: rate limit triggers 429 after threshold on /auth/login`
- `Unit: clash response excludes internal-only fields`

#### 8.3 — Performance & scale
**What**: Validate large-model behaviour and apply suggestion-1 scaling tactics.

**Design**:
- Chunked broad-phase already in 3.2; add optional partitioning of `clashes` by `clash_test_id` and `bim_elements` by `model_version_id` (suggestion-1 growth-phase DDL); materialised views for dashboard aggregates; load fixture of a 100k-element model.

**Testing**:
- `Performance (slow): 100k-element model parse < 10 min; full clash run < 5 min on reference hardware`
- `Integration: partitioned clashes table queries return identical results to unpartitioned`

#### 8.4 — Release packaging & docs
**What**: Versioned Docker images, deploy docs, seed/demo data, OpenAPI publication.

**Design**:
- Tagged multi-arch images; `docker-compose.yml` production profile; `MIGRATIONS` run on startup; admin bootstrap command; published OpenAPI 3.1 spec + generated client; AGPL-3.0 LICENSE; CONTRIBUTING.md.

**Testing**:
- `E2E (smoke): fresh compose up → seed demo project → run clash test → view in UI → export BCF, all green`
- `Integration: alembic upgrade head on empty DB succeeds; downgrade base reverses cleanly`

### Definition of Done
Partner sync works against mocks; cross-tenant access blocked everywhere; large-model targets met; tagged images build multi-arch; full smoke e2e passes; OpenAPI + license + contributing published.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Scaffolding        ─── required by everything
    │
Phase 2: Model Ingestion & IFC Parsing   ─── requires 1
    │
Phase 3: Clash Detection Engine (core)   ─── requires 2
    │
    ├── Phase 4: Issue Lifecycle & BCF    ─── requires 3 ┐ can be developed
    └── Phase 5: Browser Viewer/Frontend  ─── requires 3 ┘ concurrently
              │
Phase 6: Reporting, Analytics, Re-detect ─── requires 3 (UI surfaces need 5)
    │
Phase 7: AI-Augmented Coordination       ─── requires 3 (training data); UI needs 5
    │
Phase 8: Integrations, Hardening, Release─── requires 4 (BCF) + 6 + 7
```

**Parallelism opportunities**
- Phases **4** and **5** can be built concurrently once Phase 3 lands (one developer on BCF/issues, one on the viewer/SPA).
- Within Phase 7, **7.1**, **7.2**, **7.3** are independent and parallelisable once feature extraction (7.1 design) is agreed.
- Phase **6** reporting (6.1) can start as soon as Phase 3 data exists, before the frontend is complete.

---

## Definition of Done (per phase)

Every phase must satisfy:

1. All tasks implemented.
2. All unit and integration tests pass; designated slow/perf tests run in CI nightly.
3. `ruff check` and `ruff format --check` pass (backend); ESLint/Prettier pass (frontend).
4. `mypy` (strict) passes on `app/`; `tsc --noEmit` passes on frontend.
5. `docker build` succeeds for affected images; `docker compose config` validates.
6. The phase feature works end-to-end against the sample IFC fixtures.
7. New config options documented in `README.md` / `.env.example`.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec; BCF endpoints validate against BCF v3.0 JSON Schema.
9. Alembic migration(s) created for any schema change and `upgrade head` / `downgrade` verified on an empty DB.
10. OWASP object-level authorisation enforced on every new project-scoped route (verified by the cross-tenant test sweep once Phase 8.2 exists; manually before then).
```
