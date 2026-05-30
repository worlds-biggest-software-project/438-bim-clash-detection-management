# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL + PostGIS)

## Overview

A fully normalized relational model using PostgreSQL with the PostGIS extension for 3D spatial operations. This approach maps the BIM clash detection domain into traditional third-normal-form tables, leveraging PostgreSQL's mature ecosystem for ACID transactions, complex joins, and spatial indexing. PostGIS provides native 3D geometry types and spatial predicates (ST_3DIntersects, ST_3DDWithin) that directly support hard and soft clash detection queries.

## Design Philosophy

The BIM clash detection domain has well-defined entities with stable relationships: projects contain models, models contain elements, elements have geometry, clash tests compare element sets, and clashes flow through a resolution lifecycle. This structural stability makes a normalized relational model a natural fit. The schema below normalizes to 3NF while making pragmatic concessions for performance (e.g., materialized geometry columns, denormalized element counts on summary tables).

---

## Complete Schema Definition

### Core Identity and Multi-Tenancy

```sql
-- ============================================================
-- ORGANIZATION AND USER MANAGEMENT
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    max_projects    INTEGER NOT NULL DEFAULT 5,
    max_storage_gb  NUMERIC(10,2) NOT NULL DEFAULT 10.0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- local, oidc, saml
    auth_provider_id VARCHAR(255),
    avatar_url      VARCHAR(500),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- owner, admin, member, viewer
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_org_memberships_user ON organization_memberships(user_id);
CREATE INDEX idx_org_memberships_org ON organization_memberships(organization_id);
```

### Project Management

```sql
-- ============================================================
-- PROJECT HIERARCHY
-- ============================================================

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),  -- short project code e.g. "HOS-2026"
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'active', -- active, archived, deleted
    project_type    VARCHAR(100), -- hospital, airport, data_center, commercial, residential
    location        VARCHAR(500),
    coordinate_system VARCHAR(100), -- EPSG code or project coordinate description
    units           VARCHAR(20) NOT NULL DEFAULT 'meters', -- meters, feet, millimeters
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE project_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- coordinator, lead, member, viewer
    discipline      VARCHAR(100), -- architectural, structural, mep, civil
    notifications   BOOLEAN NOT NULL DEFAULT TRUE,
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, user_id)
);

CREATE INDEX idx_project_memberships_project ON project_memberships(project_id);
CREATE INDEX idx_project_memberships_user ON project_memberships(user_id);

-- Discipline definitions for the project
CREATE TABLE disciplines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL, -- Architectural, Structural, Mechanical, Electrical, Plumbing, Fire Protection
    abbreviation    VARCHAR(10) NOT NULL,  -- ARC, STR, MEC, ELE, PLU, FPR
    color           VARCHAR(7) NOT NULL DEFAULT '#3B82F6', -- hex color for UI
    sort_order      INTEGER NOT NULL DEFAULT 0,
    UNIQUE(project_id, abbreviation)
);
```

### BIM Model Federation

```sql
-- ============================================================
-- MODEL MANAGEMENT AND FEDERATION
-- ============================================================

CREATE TABLE model_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    discipline_id   UUID NOT NULL REFERENCES disciplines(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    source_application VARCHAR(100), -- Revit 2025, ArchiCAD 27, Tekla Structures 2024
    source_format   VARCHAR(50) NOT NULL, -- IFC2x3, IFC4, IFC4x3, NWC, RVT
    ifc_schema_version VARCHAR(20), -- IFC2X3, IFC4, IFC4X3_ADD2
    file_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL, -- S3/MinIO path to original file
    checksum_sha256 VARCHAR(64) NOT NULL,
    upload_status   VARCHAR(50) NOT NULL DEFAULT 'uploading', -- uploading, processing, ready, failed
    processing_error TEXT,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_model_files_project ON model_files(project_id);
CREATE INDEX idx_model_files_discipline ON model_files(discipline_id);

-- Each upload creates a new version; previous versions are retained
CREATE TABLE model_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_file_id   UUID NOT NULL REFERENCES model_files(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    checksum_sha256 VARCHAR(64) NOT NULL,
    ifc_project_guid VARCHAR(64), -- IfcProject GlobalId from the IFC file
    ifc_project_name VARCHAR(255),
    element_count   INTEGER NOT NULL DEFAULT 0,
    processing_status VARCHAR(50) NOT NULL DEFAULT 'pending',
    processing_started_at TIMESTAMPTZ,
    processing_completed_at TIMESTAMPTZ,
    processing_log  TEXT,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(model_file_id, version_number)
);

CREATE INDEX idx_model_versions_model ON model_versions(model_file_id);
CREATE INDEX idx_model_versions_status ON model_versions(processing_status);

-- Federated model: a named grouping of specific model versions for coordination
CREATE TABLE federated_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE federated_model_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    federated_model_id UUID NOT NULL REFERENCES federated_models(id) ON DELETE CASCADE,
    model_version_id UUID NOT NULL REFERENCES model_versions(id),
    transform_matrix DOUBLE PRECISION[16], -- 4x4 transformation matrix for alignment
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE(federated_model_id, model_version_id)
);
```

### BIM Element Storage

```sql
-- ============================================================
-- BIM ELEMENTS (parsed from IFC files)
-- ============================================================

CREATE TABLE bim_elements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    ifc_global_id   VARCHAR(64) NOT NULL, -- IFC GlobalId (22-char base64 encoded GUID)
    ifc_entity_type VARCHAR(100) NOT NULL, -- IfcWall, IfcBeam, IfcDuctSegment, IfcPipeSegment, etc.
    ifc_name        VARCHAR(500),
    ifc_description TEXT,
    ifc_object_type VARCHAR(255), -- IfcObjectType attribute (type classification)
    ifc_tag         VARCHAR(255), -- user-defined tag from authoring tool

    -- Spatial containment (parsed from IfcRelContainedInSpatialStructure)
    building_name   VARCHAR(255),
    storey_name     VARCHAR(255),
    space_name      VARCHAR(255),

    -- Classification references (parsed from IfcRelAssociatesClassification)
    classification_system VARCHAR(100), -- Uniclass, OmniClass, MasterFormat
    classification_code   VARCHAR(100),
    classification_name   VARCHAR(255),

    -- Material (parsed from IfcRelAssociatesMaterial)
    material_name   VARCHAR(255),

    -- Bounding box (axis-aligned, in project coordinates)
    bbox_min_x      DOUBLE PRECISION,
    bbox_min_y      DOUBLE PRECISION,
    bbox_min_z      DOUBLE PRECISION,
    bbox_max_x      DOUBLE PRECISION,
    bbox_max_y      DOUBLE PRECISION,
    bbox_max_z      DOUBLE PRECISION,

    -- PostGIS 3D geometry for spatial queries
    -- Stored as the bounding box polyhedron for fast intersection checks
    bbox_geom       GEOMETRY(PolyhedralSurfaceZ, 0),

    -- Volume and surface area (computed from tessellated geometry)
    volume_m3       DOUBLE PRECISION,
    surface_area_m2 DOUBLE PRECISION,

    -- Center point for proximity queries
    centroid        GEOMETRY(PointZ, 0),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Critical indexes for clash detection performance
CREATE INDEX idx_bim_elements_model_version ON bim_elements(model_version_id);
CREATE INDEX idx_bim_elements_ifc_type ON bim_elements(ifc_entity_type);
CREATE INDEX idx_bim_elements_storey ON bim_elements(storey_name);
CREATE INDEX idx_bim_elements_ifc_global_id ON bim_elements(model_version_id, ifc_global_id);
CREATE INDEX idx_bim_elements_bbox_geom ON bim_elements USING GIST(bbox_geom);
CREATE INDEX idx_bim_elements_centroid ON bim_elements USING GIST(centroid);

-- Detailed tessellated geometry stored separately (large data)
CREATE TABLE bim_element_geometries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bim_element_id  UUID NOT NULL REFERENCES bim_elements(id) ON DELETE CASCADE,
    representation_type VARCHAR(50) NOT NULL, -- tessellated, brep, extruded
    lod             VARCHAR(20) NOT NULL DEFAULT 'full', -- full, simplified, bounding_box
    vertices        BYTEA NOT NULL, -- packed float32 array [x1,y1,z1, x2,y2,z2, ...]
    indices         BYTEA NOT NULL, -- packed uint32 triangle index array
    vertex_count    INTEGER NOT NULL,
    triangle_count  INTEGER NOT NULL,
    file_format     VARCHAR(20) NOT NULL DEFAULT 'binary', -- binary, glb, draco
    storage_path    VARCHAR(500), -- external storage path for large meshes
    UNIQUE(bim_element_id, lod)
);

CREATE INDEX idx_bim_element_geometries_element ON bim_element_geometries(bim_element_id);

-- Property sets extracted from IFC (IfcPropertySet / IfcElementQuantity)
CREATE TABLE bim_element_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bim_element_id  UUID NOT NULL REFERENCES bim_elements(id) ON DELETE CASCADE,
    property_set    VARCHAR(255) NOT NULL, -- Pset_WallCommon, Qto_WallBaseQuantities, etc.
    property_name   VARCHAR(255) NOT NULL,
    property_value  TEXT,
    property_type   VARCHAR(50), -- string, real, integer, boolean, length, area, volume
    unit            VARCHAR(50),
    UNIQUE(bim_element_id, property_set, property_name)
);

CREATE INDEX idx_bim_element_props_element ON bim_element_properties(bim_element_id);
CREATE INDEX idx_bim_element_props_pset ON bim_element_properties(property_set, property_name);
```

### Clash Detection Configuration

```sql
-- ============================================================
-- CLASH TEST CONFIGURATION
-- ============================================================

CREATE TABLE clash_test_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_global       BOOLEAN NOT NULL DEFAULT FALSE, -- organization-wide template
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (
        (is_global = TRUE AND organization_id IS NOT NULL AND project_id IS NULL) OR
        (is_global = FALSE AND project_id IS NOT NULL)
    )
);

CREATE TABLE clash_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    federated_model_id UUID NOT NULL REFERENCES federated_models(id),
    template_id     UUID REFERENCES clash_test_templates(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    test_type       VARCHAR(20) NOT NULL, -- hard, soft, clearance, workflow_4d
    status          VARCHAR(50) NOT NULL DEFAULT 'draft', -- draft, running, completed, failed

    -- Selection Set A
    set_a_name      VARCHAR(255) NOT NULL DEFAULT 'Selection A',
    set_a_discipline_id UUID REFERENCES disciplines(id),
    set_a_model_version_id UUID REFERENCES model_versions(id),
    set_a_ifc_types TEXT[], -- array of IFC entity types e.g. {'IfcDuctSegment','IfcPipeSegment'}
    set_a_storey_filter TEXT[], -- filter by storey names
    set_a_search_query TEXT, -- optional free-text filter on element properties

    -- Selection Set B
    set_b_name      VARCHAR(255) NOT NULL DEFAULT 'Selection B',
    set_b_discipline_id UUID REFERENCES disciplines(id),
    set_b_model_version_id UUID REFERENCES model_versions(id),
    set_b_ifc_types TEXT[],
    set_b_storey_filter TEXT[],
    set_b_search_query TEXT,

    -- Tolerance settings
    tolerance_mm    DOUBLE PRECISION NOT NULL DEFAULT 0.0, -- hard clash: 0, soft: configurable
    clearance_mm    DOUBLE PRECISION, -- for soft/clearance tests

    -- Self-intersection settings
    allow_self_intersection BOOLEAN NOT NULL DEFAULT FALSE,
    exclude_same_discipline BOOLEAN NOT NULL DEFAULT FALSE,

    -- Execution metadata
    last_run_at     TIMESTAMPTZ,
    last_run_duration_ms INTEGER,
    total_clashes   INTEGER DEFAULT 0,
    new_clashes     INTEGER DEFAULT 0,
    active_clashes  INTEGER DEFAULT 0,
    resolved_clashes INTEGER DEFAULT 0,

    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clash_tests_project ON clash_tests(project_id);
CREATE INDEX idx_clash_tests_federated ON clash_tests(federated_model_id);
CREATE INDEX idx_clash_tests_status ON clash_tests(status);

-- Rules for ignoring known acceptable clashes
CREATE TABLE clash_ignore_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id) ON DELETE CASCADE,
    rule_type       VARCHAR(50) NOT NULL, -- ifc_type_pair, element_pair, property_match, spatial_zone
    description     TEXT,

    -- For ifc_type_pair rules: ignore clashes between specific element types
    ignore_ifc_type_a VARCHAR(100),
    ignore_ifc_type_b VARCHAR(100),

    -- For element_pair rules: ignore specific element combinations
    ignore_element_a_global_id VARCHAR(64),
    ignore_element_b_global_id VARCHAR(64),

    -- For property_match rules: ignore elements with matching property values
    ignore_property_set VARCHAR(255),
    ignore_property_name VARCHAR(255),
    ignore_property_value TEXT,

    -- For spatial_zone rules: ignore clashes in specific zones
    ignore_storey    VARCHAR(255),
    ignore_zone_bbox GEOMETRY(PolygonZ, 0),

    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clash_ignore_rules_test ON clash_ignore_rules(clash_test_id);
```

### Clash Test Runs and Results

```sql
-- ============================================================
-- CLASH TEST EXECUTION AND RESULTS
-- ============================================================

CREATE TABLE clash_test_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id) ON DELETE CASCADE,
    run_number      INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, running, completed, failed, cancelled
    triggered_by    VARCHAR(50) NOT NULL DEFAULT 'manual', -- manual, model_upload, scheduled
    triggered_by_user UUID REFERENCES users(id),

    -- Snapshot of which model versions were tested
    federated_model_snapshot JSONB NOT NULL, -- [{model_version_id, model_name, version_number}]

    -- Results summary
    total_clashes       INTEGER DEFAULT 0,
    new_clashes         INTEGER DEFAULT 0,
    active_clashes      INTEGER DEFAULT 0,
    reviewed_clashes    INTEGER DEFAULT 0,
    resolved_clashes    INTEGER DEFAULT 0,
    approved_clashes    INTEGER DEFAULT 0,
    ignored_clashes     INTEGER DEFAULT 0,

    -- Comparison with previous run
    clashes_appeared    INTEGER DEFAULT 0,
    clashes_disappeared INTEGER DEFAULT 0,
    clashes_unchanged   INTEGER DEFAULT 0,

    -- Performance metrics
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    duration_ms         INTEGER,
    elements_checked_a  INTEGER DEFAULT 0,
    elements_checked_b  INTEGER DEFAULT 0,
    pairs_evaluated     BIGINT DEFAULT 0,

    error_message       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(clash_test_id, run_number)
);

CREATE INDEX idx_clash_test_runs_test ON clash_test_runs(clash_test_id);
CREATE INDEX idx_clash_test_runs_status ON clash_test_runs(status);

-- Individual clash results
CREATE TABLE clashes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_test_run_id UUID NOT NULL REFERENCES clash_test_runs(id) ON DELETE CASCADE,
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id), -- denormalized for query performance

    -- Clash identification (stable across runs for tracking)
    clash_hash      VARCHAR(64) NOT NULL, -- SHA-256 of (element_a_global_id + element_b_global_id + test_id)
    clash_number    INTEGER NOT NULL, -- sequential within the test run

    -- Clashing elements
    element_a_id    UUID NOT NULL REFERENCES bim_elements(id),
    element_b_id    UUID NOT NULL REFERENCES bim_elements(id),
    element_a_global_id VARCHAR(64) NOT NULL,
    element_b_global_id VARCHAR(64) NOT NULL,
    element_a_ifc_type VARCHAR(100) NOT NULL,
    element_b_ifc_type VARCHAR(100) NOT NULL,
    element_a_name  VARCHAR(500),
    element_b_name  VARCHAR(500),

    -- Clash geometry
    clash_point     GEOMETRY(PointZ, 0), -- intersection point or closest approach point
    clash_distance_mm DOUBLE PRECISION, -- negative for penetration, positive for clearance violation
    intersection_volume_mm3 DOUBLE PRECISION, -- volume of intersection region (hard clashes)

    -- Spatial context
    storey_name     VARCHAR(255),
    zone_name       VARCHAR(255),
    grid_reference  VARCHAR(100), -- e.g. "Grid A-3, Level 2"

    -- Lifecycle status
    status          VARCHAR(50) NOT NULL DEFAULT 'new', -- new, active, reviewed, approved, resolved, ignored
    priority        VARCHAR(20) DEFAULT 'medium', -- critical, high, medium, low, none
    assigned_to     UUID REFERENCES users(id),
    due_date        DATE,
    resolution_type VARCHAR(50), -- design_change, tolerance_accepted, false_positive, deferred

    -- AI scoring (populated by ML pipeline)
    ai_relevance_score  DOUBLE PRECISION, -- 0.0 to 1.0, probability of being a true positive
    ai_severity_score   DOUBLE PRECISION, -- 0.0 to 1.0, estimated rework cost severity
    ai_cluster_id       UUID, -- group of clashes with shared root cause
    ai_suggested_resolution TEXT,

    -- Change tracking across runs
    first_seen_run_id UUID REFERENCES clash_test_runs(id),
    first_seen_at   TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clashes_test_run ON clashes(clash_test_run_id);
CREATE INDEX idx_clashes_test ON clashes(clash_test_id);
CREATE INDEX idx_clashes_hash ON clashes(clash_hash);
CREATE INDEX idx_clashes_status ON clashes(status);
CREATE INDEX idx_clashes_priority ON clashes(priority);
CREATE INDEX idx_clashes_assigned ON clashes(assigned_to);
CREATE INDEX idx_clashes_storey ON clashes(storey_name);
CREATE INDEX idx_clashes_element_a ON clashes(element_a_id);
CREATE INDEX idx_clashes_element_b ON clashes(element_b_id);
CREATE INDEX idx_clashes_point ON clashes USING GIST(clash_point);
CREATE INDEX idx_clashes_ai_relevance ON clashes(ai_relevance_score DESC);

-- Clash grouping (clusters of related clashes)
CREATE TABLE clash_groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    group_type      VARCHAR(50) NOT NULL, -- zone, level, discipline_pair, root_cause, ai_cluster
    status          VARCHAR(50) NOT NULL DEFAULT 'open', -- open, in_progress, resolved
    clash_count     INTEGER NOT NULL DEFAULT 0,
    assigned_to     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE clash_group_members (
    clash_group_id  UUID NOT NULL REFERENCES clash_groups(id) ON DELETE CASCADE,
    clash_id        UUID NOT NULL REFERENCES clashes(id) ON DELETE CASCADE,
    PRIMARY KEY (clash_group_id, clash_id)
);
```

### Issue Management and BCF Integration

```sql
-- ============================================================
-- ISSUE LIFECYCLE AND BCF COMPATIBILITY
-- ============================================================

CREATE TABLE issues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,

    -- BCF-compatible fields (maps directly to BCF Topic)
    bcf_guid        UUID NOT NULL DEFAULT gen_random_uuid(), -- BCF Topic GUID
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    topic_type      VARCHAR(100) NOT NULL DEFAULT 'Clash', -- Clash, Warning, Error, Request, Comment
    topic_status    VARCHAR(100) NOT NULL DEFAULT 'Open', -- Open, In Progress, Resolved, Closed
    priority        VARCHAR(50) DEFAULT 'Normal', -- Critical, Major, Normal, Minor, On hold
    stage           VARCHAR(100), -- Design, Coordination, Construction, Handover
    labels          TEXT[], -- BCF Labels array

    -- Assignment and dates
    assigned_to     UUID REFERENCES users(id),
    due_date        DATE,
    creation_author VARCHAR(255) NOT NULL,
    creation_date   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    modified_author VARCHAR(255),
    modified_date   TIMESTAMPTZ,

    -- Relationship to clashes (an issue may be created from one or more clashes)
    source_clash_id UUID REFERENCES clashes(id),
    source_clash_test_id UUID REFERENCES clash_tests(id),

    -- BCF cross-reference
    reference_links TEXT[], -- external URLs referenced in BCF
    server_assigned_id VARCHAR(50), -- human-readable sequential ID e.g. "ISS-0042"

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_issues_project ON issues(project_id);
CREATE INDEX idx_issues_status ON issues(topic_status);
CREATE INDEX idx_issues_assigned ON issues(assigned_to);
CREATE INDEX idx_issues_bcf_guid ON issues(bcf_guid);
CREATE INDEX idx_issues_source_clash ON issues(source_clash_id);

-- BCF-compatible viewpoints
CREATE TABLE issue_viewpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    issue_id        UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
    bcf_guid        UUID NOT NULL DEFAULT gen_random_uuid(),
    index_order     INTEGER NOT NULL DEFAULT 0,

    -- Camera definition
    camera_type     VARCHAR(20) NOT NULL DEFAULT 'perspective', -- perspective, orthogonal
    camera_view_point_x DOUBLE PRECISION NOT NULL,
    camera_view_point_y DOUBLE PRECISION NOT NULL,
    camera_view_point_z DOUBLE PRECISION NOT NULL,
    camera_direction_x  DOUBLE PRECISION NOT NULL,
    camera_direction_y  DOUBLE PRECISION NOT NULL,
    camera_direction_z  DOUBLE PRECISION NOT NULL,
    camera_up_vector_x  DOUBLE PRECISION NOT NULL,
    camera_up_vector_y  DOUBLE PRECISION NOT NULL,
    camera_up_vector_z  DOUBLE PRECISION NOT NULL,
    field_of_view   DOUBLE PRECISION DEFAULT 60.0,
    aspect_ratio    DOUBLE PRECISION DEFAULT 1.778,

    -- Snapshot image
    snapshot_path   VARCHAR(500), -- storage path to PNG snapshot
    snapshot_type   VARCHAR(10) DEFAULT 'png',

    -- Component visibility and selection
    default_visibility BOOLEAN NOT NULL DEFAULT TRUE,
    selected_components TEXT[], -- array of IFC GlobalIds
    exception_components TEXT[], -- array of IFC GlobalIds (visible when default=hidden, vice versa)

    -- Coloring (maps IFC GlobalIds to ARGB hex colors)
    component_coloring JSONB, -- [{"color": "#FF0000FF", "components": ["globalId1", "globalId2"]}]

    -- Clipping planes
    clipping_planes JSONB, -- [{"location": {x,y,z}, "direction": {x,y,z}}]

    -- 3D markup lines
    markup_lines    JSONB, -- [{"start": {x,y,z}, "end": {x,y,z}}]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_issue_viewpoints_issue ON issue_viewpoints(issue_id);

-- Comments on issues (BCF Comment equivalent)
CREATE TABLE issue_comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    issue_id        UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
    bcf_guid        UUID NOT NULL DEFAULT gen_random_uuid(),
    author          VARCHAR(255) NOT NULL,
    author_user_id  UUID REFERENCES users(id),
    comment_text    TEXT NOT NULL,
    viewpoint_id    UUID REFERENCES issue_viewpoints(id),
    modified_author VARCHAR(255),
    modified_date   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_issue_comments_issue ON issue_comments(issue_id);

-- File attachments on issues
CREATE TABLE issue_attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    issue_id        UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
    filename        VARCHAR(255) NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- BCF import/export log
CREATE TABLE bcf_exchanges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    direction       VARCHAR(10) NOT NULL, -- import, export
    bcf_version     VARCHAR(10) NOT NULL, -- 2.1, 3.0
    filename        VARCHAR(255) NOT NULL,
    storage_path    VARCHAR(500),
    topics_count    INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'completed',
    error_message   TEXT,
    performed_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Reporting and Analytics

```sql
-- ============================================================
-- REPORTING AND ANALYTICS
-- ============================================================

-- Daily snapshot for trend analytics
CREATE TABLE clash_daily_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    total_clashes   INTEGER NOT NULL DEFAULT 0,
    new_clashes     INTEGER NOT NULL DEFAULT 0,
    active_clashes  INTEGER NOT NULL DEFAULT 0,
    reviewed_clashes INTEGER NOT NULL DEFAULT 0,
    resolved_clashes INTEGER NOT NULL DEFAULT 0,
    ignored_clashes INTEGER NOT NULL DEFAULT 0,
    critical_count  INTEGER NOT NULL DEFAULT 0,
    high_count      INTEGER NOT NULL DEFAULT 0,
    medium_count    INTEGER NOT NULL DEFAULT 0,
    low_count       INTEGER NOT NULL DEFAULT 0,
    avg_resolution_hours DOUBLE PRECISION,
    UNIQUE(clash_test_id, snapshot_date)
);

CREATE INDEX idx_clash_snapshots_project_date ON clash_daily_snapshots(project_id, snapshot_date);

-- Saved report configurations
CREATE TABLE report_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    report_type     VARCHAR(50) NOT NULL, -- clash_summary, discipline_matrix, trend, workload
    format          VARCHAR(20) NOT NULL DEFAULT 'pdf', -- pdf, csv, xlsx
    filters         JSONB, -- stored filter configuration
    schedule_cron   VARCHAR(100), -- optional recurring schedule
    recipients      TEXT[], -- email addresses for scheduled delivery
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Audit trail
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    project_id      UUID REFERENCES projects(id),
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(100) NOT NULL, -- clash, issue, model_file, clash_test, etc.
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL, -- create, update, delete, status_change
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_project ON audit_log(project_id);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
```

### AI/ML Pipeline Support

```sql
-- ============================================================
-- AI/ML PIPELINE TABLES
-- ============================================================

-- Training data from resolved clashes for ML model training
CREATE TABLE ml_training_samples (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_id        UUID NOT NULL REFERENCES clashes(id) ON DELETE CASCADE,
    feature_vector  DOUBLE PRECISION[] NOT NULL, -- extracted features for ML
    label_relevant  BOOLEAN, -- was this a true positive?
    label_severity  VARCHAR(20), -- critical, high, medium, low
    label_resolution_type VARCHAR(50),
    resolution_time_hours DOUBLE PRECISION,
    labeled_by      UUID REFERENCES users(id),
    labeled_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ML model versions
CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_type      VARCHAR(50) NOT NULL, -- relevance_classifier, severity_predictor, cluster_model
    version         VARCHAR(50) NOT NULL,
    framework       VARCHAR(50) NOT NULL, -- scikit-learn, pytorch, xgboost
    metrics         JSONB, -- {accuracy, precision, recall, f1, auc}
    artifact_path   VARCHAR(500) NOT NULL, -- S3 path to serialized model
    training_samples_count INTEGER NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT FALSE,
    trained_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    activated_at    TIMESTAMPTZ
);

-- Natural language query log (for AI query interface)
CREATE TABLE nl_query_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    query_text      TEXT NOT NULL,
    parsed_filters  JSONB, -- structured interpretation of the natural language query
    result_count    INTEGER,
    execution_time_ms INTEGER,
    user_feedback   VARCHAR(20), -- helpful, not_helpful, null
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Key Relationships Diagram (textual)

```
Organization 1──* Project 1──* Discipline
     |                |
     |                +──* ModelFile 1──* ModelVersion 1──* BimElement 1──* BimElementProperty
     |                |                                         |
     |                +──* FederatedModel 1──* FederatedModelMember     +──1 BimElementGeometry
     |                |
     |                +──* ClashTest 1──* ClashTestRun 1──* Clash *──1 BimElement (A)
     |                |       |                                *──1 BimElement (B)
     |                |       +──* ClashIgnoreRule
     |                |
     |                +──* Issue 1──* IssueComment
     |                |        +──* IssueViewpoint
     |                |        +──* IssueAttachment
     |                |
     |                +──* ClashDailySnapshot
     |
     +──* User ──* OrganizationMembership
              +──* ProjectMembership
```

---

## Pros and Cons

### Pros

1. **Strong data integrity**: Foreign key constraints, unique constraints, and check constraints guarantee referential integrity across the entire clash lifecycle. A clash cannot reference a non-existent element; an issue cannot exist without a project.

2. **BCF standard alignment**: The issues/viewpoints/comments tables map directly to the BCF 3.0 XML schema (Topic, Viewpoint, Comment), making import/export a straightforward field-by-field mapping without lossy transformations.

3. **Mature spatial capabilities**: PostGIS provides battle-tested 3D spatial indexes (GiST R-trees) and predicates (ST_3DIntersects, ST_3DDWithin, ST_3DDistance) that can serve as a first-pass filter for clash detection before exact mesh intersection testing.

4. **Rich query capabilities**: Complex ad-hoc queries (e.g., "find all unresolved critical clashes on Level 3 between MEP and structural disciplines assigned to user X") are natural in SQL with proper indexing.

5. **Ecosystem maturity**: PostgreSQL has robust tooling for backups, replication (streaming, logical), monitoring (pg_stat_statements), connection pooling (PgBouncer), and migration management (Flyway, Alembic).

6. **Audit trail simplicity**: The audit_log table with JSONB old/new values provides a complete change history without the complexity of event sourcing.

7. **Analytics friendly**: Daily snapshot tables and standard SQL make dashboard analytics, trend analysis, and reporting straightforward with any BI tool.

### Cons

1. **Schema rigidity**: Adding new IFC property sets, custom element attributes, or project-specific metadata requires schema migrations. The BIM domain frequently introduces new entity types and properties with each IFC version.

2. **Geometry storage limitations**: While PostGIS handles bounding-box intersection well, storing and querying detailed tessellated meshes (millions of triangles) in a relational database is inefficient. The bim_element_geometries table with BYTEA columns is essentially a blob store within a relational wrapper.

3. **Scaling ceiling for large projects**: A major hospital project may have 500,000+ BIM elements across 20+ disciplines. Cross-joining two element sets for pairwise clash checking (N*M comparisons) can overwhelm PostgreSQL even with spatial indexing, requiring application-level chunking.

4. **Element property explosion**: The EAV-pattern bim_element_properties table is necessary for IFC's open property model but produces slow queries when filtering on property values. A single element may have 50+ properties, creating significant row multiplication.

5. **No native change feed**: Unlike event-sourced models, tracking what changed between clash runs requires explicit diff computation. The clashes table stores current state; reconstructing the history of a clash's status transitions requires querying the audit_log.

6. **Horizontal scaling complexity**: PostgreSQL scales vertically well but horizontal sharding (e.g., by project_id via Citus) adds operational complexity and restricts cross-shard queries.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Database | PostgreSQL 16+ with PostGIS 3.4+ | Native 3D geometry support, JSONB for semi-structured data |
| Connection pooling | PgBouncer | Essential for web application connection management |
| Migrations | Flyway or Alembic | Version-controlled schema evolution |
| Spatial indexing | GiST indexes on geometry columns | Required for performant spatial queries |
| Full-text search | PostgreSQL tsvector + GIN indexes | For natural-language search across element names/descriptions |
| Blob storage | S3/MinIO with path references | Store IFC files, tessellated geometry, snapshots externally |
| Caching | Redis | Cache clash test results, dashboard aggregations |
| Task queue | Celery + Redis/RabbitMQ | Async processing of IFC parsing and clash detection |
| ORM | SQLAlchemy (Python) or Prisma (TypeScript) | Type-safe database access with migration support |

---

## Migration and Scaling Considerations

### Initial Deployment (0-50 projects)

- Single PostgreSQL instance (8 CPU, 32GB RAM, NVMe SSD)
- All tables in a single database with schema-based isolation for testing
- PostGIS extension enabled for spatial queries
- Basic connection pooling via PgBouncer (100 connections)

### Growth Phase (50-500 projects)

- Read replicas for dashboard queries and report generation
- Partitioning of large tables:
  - `bim_elements` partitioned by `model_version_id` (hash) or by project
  - `clashes` partitioned by `clash_test_id` (hash)
  - `audit_log` partitioned by `created_at` (range, monthly)
  - `bim_element_properties` partitioned by `bim_element_id` (hash)
- Move geometry storage to dedicated object storage (S3/MinIO)
- Introduce materialized views for dashboard aggregations, refreshed on schedule

```sql
-- Example: partition clashes by clash_test_id
CREATE TABLE clashes (
    -- ... all columns as above ...
) PARTITION BY HASH (clash_test_id);

CREATE TABLE clashes_p0 PARTITION OF clashes FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE clashes_p1 PARTITION OF clashes FOR VALUES WITH (MODULUS 16, REMAINDER 1);
-- ... through p15
```

### Enterprise Scale (500+ projects, millions of elements)

- Citus extension for distributed PostgreSQL (shard by project_id)
- Dedicated analytical database (e.g., ClickHouse or TimescaleDB) for historical trend queries
- Separate read-optimized replicas per region for global teams
- Archive completed project data to cold storage after project closeout
- Consider moving BIM element geometry queries to a specialized engine (e.g., spatial index server) while keeping metadata in PostgreSQL

### Data Retention and Archival

```sql
-- Archive policy: move completed project data to archive schema after 2 years
CREATE SCHEMA archive;

-- Automated archival procedure
CREATE OR REPLACE FUNCTION archive_completed_project(p_project_id UUID)
RETURNS VOID AS $$
BEGIN
    -- Move to archive schema, preserving all data
    INSERT INTO archive.clashes SELECT * FROM public.clashes
    WHERE clash_test_id IN (SELECT id FROM clash_tests WHERE project_id = p_project_id);

    DELETE FROM public.clashes
    WHERE clash_test_id IN (SELECT id FROM clash_tests WHERE project_id = p_project_id);

    -- Repeat for other tables...
    UPDATE projects SET status = 'archived' WHERE id = p_project_id;
END;
$$ LANGUAGE plpgsql;
```

### Migration from Existing Systems

For organizations migrating from Navisworks NWD/NWF files:
1. Parse NWD clash reports (XML export) into the `clashes` table
2. Map Navisworks clash statuses to the issue lifecycle
3. Import BCF files from BIMcollab/Solibri directly into issues/viewpoints tables
4. Use IfcOpenShell to parse IFC files and populate `bim_elements` and `bim_element_properties`

For organizations using Autodesk Construction Cloud:
1. Use ACC REST API to pull clash results
2. Map ACC clash issue fields to the normalized schema
3. Bidirectional sync via webhooks for ongoing coordination
