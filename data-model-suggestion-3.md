# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

## Overview

A pragmatic hybrid approach that uses normalized PostgreSQL tables for stable, frequently queried entities (projects, users, clash tests, issue lifecycle) while leveraging PostgreSQL's native JSONB columns for inherently variable or schema-less data (IFC element properties, clash detection parameters, viewpoint definitions, AI scoring metadata). This avoids the rigidity of full normalization for BIM data that varies by IFC version, authoring tool, and discipline, while retaining relational integrity and SQL query power for the coordination workflow.

## Design Philosophy

The BIM clash detection domain has two distinct data shapes:

1. **Stable workflow data**: Projects, users, disciplines, clash tests, issue lifecycle states, and BCF-compatible fields follow predictable structures. These benefit from normalized tables with foreign keys, constraints, and indexes.

2. **Variable domain data**: IFC element properties vary wildly (a wall has fire rating properties, a duct has airflow properties, a beam has structural capacity properties). Clash detection parameters differ by test type. Viewpoint serialization includes variable camera settings, component colorings, and clipping planes. AI scoring metadata evolves as models improve. Forcing these into rigid columns creates either sparse tables or an explosion of EAV rows.

The hybrid approach stores category-1 data in normalized columns and category-2 data in JSONB columns on the same tables, getting the best of both worlds: referential integrity where it matters, flexibility where it is needed, and the ability to index into JSONB using GIN indexes for query performance.

---

## Complete Schema Definition

### Organizational Foundation

```sql
-- ============================================================
-- STABLE ENTITIES: ORGANIZATIONS, USERS, PROJECTS
-- These rarely change structure and benefit from full normalization
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "postgis";

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings schema: {
    --   "subscriptionTier": "professional",
    --   "maxProjects": 50,
    --   "maxStorageGb": 500,
    --   "defaultUnits": "meters",
    --   "branding": {"logoUrl": "...", "primaryColor": "#003366"},
    --   "integrations": {
    --     "procore": {"enabled": true, "apiKey": "encrypted:..."},
    --     "bimcollab": {"enabled": false}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile schema: {
    --   "avatar_url": "https://...",
    --   "phone": "+1...",
    --   "company": "ACME Engineering",
    --   "title": "BIM Coordinator",
    --   "timezone": "America/New_York",
    --   "notification_preferences": {
    --     "email_on_assignment": true,
    --     "email_on_clash_detected": false,
    --     "slack_webhook": "https://hooks.slack.com/..."
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    UNIQUE(organization_id, user_id)
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',

    -- Stable fields that are always present
    units           VARCHAR(20) NOT NULL DEFAULT 'meters',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Variable project metadata in JSONB
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata schema: {
    --   "projectType": "hospital",
    --   "location": {
    --     "address": "123 Main St, Springfield",
    --     "coordinates": {"lat": 39.781, "lng": -89.650},
    --     "coordinateSystem": "EPSG:3857"
    --   },
    --   "phases": ["Design Development", "Construction Documents"],
    --   "currentPhase": "Construction Documents",
    --   "client": "Springfield Health Authority",
    --   "contractValue": 45000000,
    --   "targetCompletion": "2028-06-30",
    --   "customFields": {
    --     "permitNumber": "BP-2026-4521",
    --     "leedCertification": "Gold"
    --   }
    -- }

    -- Coordination settings for clash detection
    coordination_settings JSONB NOT NULL DEFAULT '{}',
    -- coordination_settings schema: {
    --   "defaultToleranceMm": 10.0,
    --   "autoDetectOnUpload": true,
    --   "clashStatuses": ["New", "Active", "Reviewed", "Approved", "Resolved"],
    --   "clashPriorities": ["Critical", "High", "Medium", "Low"],
    --   "issueTypes": ["Clash", "Warning", "Info", "Request"],
    --   "zones": [
    --     {"name": "Wing A", "levels": ["L1", "L2", "L3"]},
    --     {"name": "Wing B", "levels": ["L1", "L2"]}
    --   ],
    --   "gridSystem": {
    --     "xGrids": ["A", "B", "C", "D", "E"],
    --     "yGrids": ["1", "2", "3", "4"]
    --   }
    -- }
);

CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_metadata ON projects USING GIN(metadata jsonb_path_ops);

CREATE TABLE project_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    discipline      VARCHAR(100),
    UNIQUE(project_id, user_id)
);

CREATE TABLE disciplines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    abbreviation    VARCHAR(10) NOT NULL,
    color           VARCHAR(7) NOT NULL DEFAULT '#3B82F6',
    sort_order      INTEGER NOT NULL DEFAULT 0,
    UNIQUE(project_id, abbreviation)
);
```

### Model Federation with JSONB Properties

```sql
-- ============================================================
-- MODEL FILES AND VERSIONS
-- Stable columns for identity/status + JSONB for IFC metadata
-- ============================================================

CREATE TABLE model_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    discipline_id   UUID NOT NULL REFERENCES disciplines(id),
    name            VARCHAR(255) NOT NULL,
    source_format   VARCHAR(50) NOT NULL, -- IFC2x3, IFC4, IFC4x3
    file_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    checksum_sha256 VARCHAR(64) NOT NULL,
    upload_status   VARCHAR(50) NOT NULL DEFAULT 'uploading',
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Variable file metadata in JSONB
    source_metadata JSONB NOT NULL DEFAULT '{}',
    -- source_metadata schema: {
    --   "sourceApplication": "Autodesk Revit 2025",
    --   "sourceApplicationVersion": "2025.1.0",
    --   "exportPlugin": "IFC Exporter 25.1.0.13",
    --   "ifcSchemaVersion": "IFC4",
    --   "modelViewDefinition": "DesignTransferView",
    --   "author": "Jane Smith",
    --   "organization": "Smith MEP Consulting",
    --   "preprocessor": "Geometry Gym IFC Library",
    --   "originatingSystem": "Revit",
    --   "authorization": "CCBIM"
    -- }
);

CREATE INDEX idx_model_files_project ON model_files(project_id);

CREATE TABLE model_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_file_id   UUID NOT NULL REFERENCES model_files(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    checksum_sha256 VARCHAR(64) NOT NULL,
    element_count   INTEGER NOT NULL DEFAULT 0,
    processing_status VARCHAR(50) NOT NULL DEFAULT 'pending',
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- IFC project-level metadata extracted during processing
    ifc_metadata    JSONB NOT NULL DEFAULT '{}',
    -- ifc_metadata schema: {
    --   "ifcProjectGuid": "0YvctVUKr0kugbFTf53O9L",
    --   "ifcProjectName": "Springfield Hospital MEP",
    --   "ifcSiteName": "Main Campus",
    --   "ifcBuildingName": "Building A",
    --   "storeys": [
    --     {"name": "Level 0 - Foundation", "elevation": 0.0},
    --     {"name": "Level 1 - Ground", "elevation": 4.2},
    --     {"name": "Level 2", "elevation": 8.4}
    --   ],
    --   "elementTypeCounts": {"IfcWall": 342, "IfcDuctSegment": 1204, "IfcPipeSegment": 876},
    --   "boundingBox": {
    --     "min": {"x": -50.0, "y": -30.0, "z": -5.0},
    --     "max": {"x": 120.0, "y": 80.0, "z": 45.0}
    --   },
    --   "processingLog": "Parsed in 12.4s. 2422 elements extracted. 14 warnings.",
    --   "warnings": ["IfcProxy entity at line 14523 skipped", "..."]
    -- }

    UNIQUE(model_file_id, version_number)
);

CREATE INDEX idx_model_versions_model ON model_versions(model_file_id);
CREATE INDEX idx_model_versions_status ON model_versions(processing_status);

CREATE TABLE federated_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Member model versions with their transform matrices
    member_models   JSONB NOT NULL DEFAULT '[]'
    -- member_models schema: [
    --   {
    --     "modelVersionId": "uuid",
    --     "modelFileName": "MEP_Level2.ifc",
    --     "disciplineName": "Mechanical",
    --     "transformMatrix": [1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1],
    --     "isVisible": true
    --   }
    -- ]
);

CREATE INDEX idx_federated_models_project ON federated_models(project_id);
```

### BIM Elements: Normalized Core + JSONB Properties

```sql
-- ============================================================
-- BIM ELEMENTS
-- Normalized: identity, type, spatial containment, bounding box
-- JSONB: full property sets, classification details, material info
-- ============================================================

CREATE TABLE bim_elements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,

    -- Normalized core attributes (queried in every clash detection operation)
    ifc_global_id   VARCHAR(64) NOT NULL,
    ifc_entity_type VARCHAR(100) NOT NULL,
    ifc_name        VARCHAR(500),

    -- Spatial containment (critical for filtering and grouping)
    building_name   VARCHAR(255),
    storey_name     VARCHAR(255),
    space_name      VARCHAR(255),

    -- Bounding box (used in spatial intersection queries)
    bbox_min_x      DOUBLE PRECISION NOT NULL,
    bbox_min_y      DOUBLE PRECISION NOT NULL,
    bbox_min_z      DOUBLE PRECISION NOT NULL,
    bbox_max_x      DOUBLE PRECISION NOT NULL,
    bbox_max_y      DOUBLE PRECISION NOT NULL,
    bbox_max_z      DOUBLE PRECISION NOT NULL,

    -- PostGIS geometry for spatial queries
    bbox_geom       GEOMETRY(PolyhedralSurfaceZ, 0),
    centroid        GEOMETRY(PointZ, 0),

    -- Computed metrics
    volume_m3       DOUBLE PRECISION,
    surface_area_m2 DOUBLE PRECISION,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- ================================================================
    -- JSONB: All variable IFC properties, classifications, materials
    -- This is where the hybrid approach pays off. A single element
    -- might have 5 property sets with 50+ properties. Rather than
    -- an EAV table with 50 rows per element (and 50x row count for
    -- 500K elements = 25M rows), we store them as structured JSONB.
    -- ================================================================

    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties schema: {
    --   "Pset_WallCommon": {
    --     "IsExternal": {"value": true, "type": "boolean"},
    --     "FireRating": {"value": "2HR", "type": "string"},
    --     "ThermalTransmittance": {"value": 0.24, "type": "real", "unit": "W/(m2·K)"},
    --     "LoadBearing": {"value": true, "type": "boolean"},
    --     "AcousticRating": {"value": "STC 55", "type": "string"}
    --   },
    --   "Qto_WallBaseQuantities": {
    --     "Length": {"value": 8.5, "type": "length", "unit": "m"},
    --     "Height": {"value": 3.2, "type": "length", "unit": "m"},
    --     "Width": {"value": 0.2, "type": "length", "unit": "m"},
    --     "GrossArea": {"value": 27.2, "type": "area", "unit": "m2"},
    --     "NetVolume": {"value": 5.44, "type": "volume", "unit": "m3"}
    --   },
    --   "Pset_ManufacturerTypeInformation": {
    --     "Manufacturer": {"value": "USG", "type": "string"},
    --     "ModelReference": {"value": "Sheetrock Brand X", "type": "string"}
    --   }
    -- }

    classification  JSONB NOT NULL DEFAULT '{}',
    -- classification schema: {
    --   "system": "Uniclass 2015",
    --   "code": "Ss_25_10_30",
    --   "name": "Masonry wall systems",
    --   "additionalClassifications": [
    --     {"system": "OmniClass", "code": "23-13 11 00", "name": "Concrete Walls"}
    --   ]
    -- }

    materials       JSONB NOT NULL DEFAULT '{}',
    -- materials schema: {
    --   "primary": "Concrete, Cast-in-Place",
    --   "layers": [
    --     {"name": "Concrete", "thickness_mm": 200, "material": "C30/37"},
    --     {"name": "Insulation", "thickness_mm": 100, "material": "Rockwool"},
    --     {"name": "Plasterboard", "thickness_mm": 12.5, "material": "Gypsum"}
    --   ]
    -- }

    -- Authoring tool-specific metadata that does not map to IFC standards
    custom_metadata JSONB NOT NULL DEFAULT '{}'
    -- custom_metadata schema: {
    --   "revitElementId": 1234567,
    --   "revitCategory": "Walls",
    --   "revitFamily": "Basic Wall",
    --   "revitType": "Generic - 200mm",
    --   "revitWorkset": "Shared Levels and Grids",
    --   "revitPhaseCreated": "New Construction",
    --   "teklaPartMark": "B-123"
    -- }
);

-- Normalized column indexes (used in every query)
CREATE INDEX idx_bim_elements_model_version ON bim_elements(model_version_id);
CREATE INDEX idx_bim_elements_ifc_type ON bim_elements(ifc_entity_type);
CREATE INDEX idx_bim_elements_storey ON bim_elements(storey_name);
CREATE INDEX idx_bim_elements_global_id ON bim_elements(model_version_id, ifc_global_id);
CREATE INDEX idx_bim_elements_bbox ON bim_elements USING GIST(bbox_geom);
CREATE INDEX idx_bim_elements_centroid ON bim_elements USING GIST(centroid);

-- GIN indexes on JSONB for property-based filtering
CREATE INDEX idx_bim_elements_properties ON bim_elements USING GIN(properties jsonb_path_ops);
CREATE INDEX idx_bim_elements_classification ON bim_elements USING GIN(classification jsonb_path_ops);
CREATE INDEX idx_bim_elements_materials ON bim_elements USING GIN(materials jsonb_path_ops);
CREATE INDEX idx_bim_elements_custom ON bim_elements USING GIN(custom_metadata jsonb_path_ops);

-- Geometry stored separately (large binary data)
CREATE TABLE bim_element_geometries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bim_element_id  UUID NOT NULL REFERENCES bim_elements(id) ON DELETE CASCADE,
    lod             VARCHAR(20) NOT NULL DEFAULT 'full',
    format          VARCHAR(20) NOT NULL DEFAULT 'glb', -- glb, draco, binary
    data_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,  -- external object storage path
    vertex_count    INTEGER NOT NULL,
    triangle_count  INTEGER NOT NULL,

    -- Inline mesh metadata for the viewer
    mesh_metadata   JSONB NOT NULL DEFAULT '{}',
    -- mesh_metadata schema: {
    --   "boundingSphere": {"center": [0,0,0], "radius": 5.2},
    --   "materialIds": ["mat_concrete_gray", "mat_steel_galvanized"],
    --   "hasTextures": false,
    --   "compressionRatio": 0.12
    -- }

    UNIQUE(bim_element_id, lod)
);
```

### Clash Detection Configuration

```sql
-- ============================================================
-- CLASH TEST CONFIGURATION
-- Mix of normalized columns for filterable fields
-- and JSONB for complex, variable configuration
-- ============================================================

CREATE TABLE clash_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    federated_model_id UUID NOT NULL REFERENCES federated_models(id),
    name            VARCHAR(255) NOT NULL,
    test_type       VARCHAR(20) NOT NULL, -- hard, soft, clearance, workflow_4d
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',

    -- Tolerance (normalized because it is filtered and sorted on)
    tolerance_mm    DOUBLE PRECISION NOT NULL DEFAULT 0.0,
    clearance_mm    DOUBLE PRECISION,

    -- Result summary (denormalized for list views)
    total_clashes   INTEGER DEFAULT 0,
    new_clashes     INTEGER DEFAULT 0,
    active_clashes  INTEGER DEFAULT 0,
    resolved_clashes INTEGER DEFAULT 0,
    last_run_at     TIMESTAMPTZ,
    last_run_number INTEGER DEFAULT 0,

    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- ================================================================
    -- JSONB: Complex selection set definitions
    -- These vary significantly: some tests filter by IFC type,
    -- others by property values, others by spatial zone.
    -- JSONB handles this variability elegantly.
    -- ================================================================

    selection_set_a JSONB NOT NULL DEFAULT '{}',
    -- selection_set_a schema: {
    --   "name": "MEP Ductwork",
    --   "disciplineId": "uuid",
    --   "modelVersionId": "uuid",
    --   "ifcTypes": ["IfcDuctSegment", "IfcDuctFitting", "IfcFlowTerminal"],
    --   "storeyFilter": ["Level 2", "Level 3"],
    --   "propertyFilters": [
    --     {"pset": "Pset_DistributionSystemCommon", "property": "SystemType", "operator": "eq", "value": "HVAC"},
    --     {"pset": "Qto_DuctSegmentBaseQuantities", "property": "Length", "operator": "gt", "value": 0.5}
    --   ],
    --   "spatialFilter": {
    --     "type": "bbox",
    --     "min": {"x": 0, "y": 0, "z": 0},
    --     "max": {"x": 50, "y": 30, "z": 15}
    --   },
    --   "excludeGlobalIds": ["0YvctVUKr0kugbFTf53O9L"]
    -- }

    selection_set_b JSONB NOT NULL DEFAULT '{}',

    -- Ignore rules (complex, variable criteria)
    ignore_rules    JSONB NOT NULL DEFAULT '[]',
    -- ignore_rules schema: [
    --   {
    --     "id": "uuid",
    --     "ruleType": "ifc_type_pair",
    --     "description": "Ignore duct-insulation clashes (expected overlap)",
    --     "ifcTypeA": "IfcDuctSegment",
    --     "ifcTypeB": "IfcCovering",
    --     "isActive": true,
    --     "createdBy": "uuid",
    --     "createdAt": "2026-01-15T10:00:00Z"
    --   },
    --   {
    --     "id": "uuid",
    --     "ruleType": "property_match",
    --     "description": "Ignore clashes between elements in same system",
    --     "matchProperty": {"pset": "Pset_DistributionSystemCommon", "property": "SystemName"},
    --     "isActive": true
    --   },
    --   {
    --     "id": "uuid",
    --     "ruleType": "spatial_zone",
    --     "description": "Ignore shaft area (expected penetrations)",
    --     "zone": {"min": {"x": 10, "y": 20, "z": 0}, "max": {"x": 15, "y": 25, "z": 50}},
    --     "isActive": true
    --   }
    -- ]

    -- AI configuration for this test
    ai_config       JSONB NOT NULL DEFAULT '{}'
    -- ai_config schema: {
    --   "enableRelevanceScoring": true,
    --   "relevanceThreshold": 0.3,
    --   "enableSeverityPrediction": true,
    --   "enableClustering": true,
    --   "enableResolutionSuggestions": true,
    --   "modelVersionOverrides": {
    --     "relevance": "v2.3.1",
    --     "severity": "v1.8.0"
    --   }
    -- }
);

CREATE INDEX idx_clash_tests_project ON clash_tests(project_id);
CREATE INDEX idx_clash_tests_status ON clash_tests(status);
CREATE INDEX idx_clash_tests_federated ON clash_tests(federated_model_id);
```

### Clash Test Runs and Results

```sql
-- ============================================================
-- CLASH TEST RUNS
-- ============================================================

CREATE TABLE clash_test_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id) ON DELETE CASCADE,
    run_number      INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    triggered_by    VARCHAR(50) NOT NULL DEFAULT 'manual',
    triggered_by_user UUID REFERENCES users(id),

    -- Result counts (normalized for dashboard queries)
    total_clashes       INTEGER DEFAULT 0,
    new_clashes         INTEGER DEFAULT 0,
    active_clashes      INTEGER DEFAULT 0,
    resolved_clashes    INTEGER DEFAULT 0,
    ignored_clashes     INTEGER DEFAULT 0,
    clashes_appeared    INTEGER DEFAULT 0,
    clashes_disappeared INTEGER DEFAULT 0,

    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    duration_ms         INTEGER,

    -- Variable execution metadata in JSONB
    execution_context   JSONB NOT NULL DEFAULT '{}',
    -- execution_context schema: {
    --   "federatedModelSnapshot": [
    --     {"modelVersionId": "uuid", "modelName": "MEP_L2.ifc", "versionNumber": 3}
    --   ],
    --   "elementsCheckedA": 1204,
    --   "elementsCheckedB": 342,
    --   "pairsEvaluated": 411768,
    --   "spatialIndexBuildTimeMs": 450,
    --   "intersectionTestTimeMs": 8200,
    --   "aiScoringTimeMs": 1200,
    --   "workerCount": 4,
    --   "peakMemoryMb": 2400,
    --   "errors": [],
    --   "warnings": ["3 elements had degenerate geometry, skipped"]
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(clash_test_id, run_number)
);

CREATE INDEX idx_clash_test_runs_test ON clash_test_runs(clash_test_id);
CREATE INDEX idx_clash_test_runs_status ON clash_test_runs(status);

-- ============================================================
-- CLASHES (core results)
-- Normalized: identity, status, spatial location, element refs
-- JSONB: AI scores, extended geometry details, resolution metadata
-- ============================================================

CREATE TABLE clashes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_test_run_id UUID NOT NULL REFERENCES clash_test_runs(id) ON DELETE CASCADE,
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id),

    -- Stable identity
    clash_hash      VARCHAR(64) NOT NULL,
    clash_number    INTEGER NOT NULL,

    -- Element references (normalized for joins)
    element_a_id    UUID NOT NULL REFERENCES bim_elements(id),
    element_b_id    UUID NOT NULL REFERENCES bim_elements(id),
    element_a_global_id VARCHAR(64) NOT NULL,
    element_b_global_id VARCHAR(64) NOT NULL,
    element_a_ifc_type VARCHAR(100) NOT NULL,
    element_b_ifc_type VARCHAR(100) NOT NULL,

    -- Clash point (normalized for spatial queries and map views)
    clash_point     GEOMETRY(PointZ, 0),
    distance_mm     DOUBLE PRECISION,

    -- Spatial context (normalized for grouping and filtering)
    storey_name     VARCHAR(255),
    zone_name       VARCHAR(255),
    grid_reference  VARCHAR(100),

    -- Lifecycle status (normalized for status-based queries)
    status          VARCHAR(50) NOT NULL DEFAULT 'new',
    priority        VARCHAR(20) DEFAULT 'medium',
    assigned_to     UUID REFERENCES users(id),
    due_date        DATE,

    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- ================================================================
    -- JSONB: Variable/extended clash data
    -- ================================================================

    -- Detailed element context (denormalized from bim_elements for display)
    element_a_details JSONB NOT NULL DEFAULT '{}',
    -- element_a_details schema: {
    --   "name": "Rectangular Duct - 600x400",
    --   "discipline": "Mechanical",
    --   "modelName": "MEP_Level2.ifc",
    --   "material": "Galvanized Steel",
    --   "systemName": "AHU-2 Supply",
    --   "classification": {"system": "Uniclass", "code": "Ss_60_40_33"},
    --   "keyProperties": {
    --     "Width": "600mm",
    --     "Height": "400mm",
    --     "InsulationThickness": "25mm"
    --   }
    -- }

    element_b_details JSONB NOT NULL DEFAULT '{}',

    -- Clash geometry details
    clash_geometry  JSONB NOT NULL DEFAULT '{}',
    -- clash_geometry schema: {
    --   "intersectionVolumeMm3": 145230.5,
    --   "intersectionBbox": {
    --     "min": {"x": 12.45, "y": 8.32, "z": 6.10},
    --     "max": {"x": 12.85, "y": 8.72, "z": 6.55}
    --   },
    --   "penetrationDepthMm": 42.3,
    --   "penetrationVector": {"x": 0.0, "y": 0.0, "z": -1.0},
    --   "contactSurfaceAreaMm2": 24000.0
    -- }

    -- AI analysis results
    ai_analysis     JSONB NOT NULL DEFAULT '{}',
    -- ai_analysis schema: {
    --   "relevanceScore": 0.87,
    --   "severityScore": 0.72,
    --   "clusterId": "uuid",
    --   "clusterName": "AHU-2 duct routing conflicts at Level 2",
    --   "suggestedResolution": "Reroute duct below beam B-234 with 50mm offset",
    --   "similarPastClashes": [
    --     {"projectId": "uuid", "clashHash": "abc123", "resolution": "Lowered duct by 100mm", "similarity": 0.92}
    --   ],
    --   "predictedReworkCost": {"min": 2500, "max": 8000, "currency": "USD"},
    --   "modelVersion": "relevance_v2.3.1",
    --   "scoredAt": "2026-01-15T10:30:00Z"
    -- }

    -- Resolution metadata (filled when resolved)
    resolution      JSONB NOT NULL DEFAULT '{}',
    -- resolution schema: {
    --   "type": "design_change",
    --   "notes": "MEP contractor rerouted duct below beam per RFI-234",
    --   "rfiReference": "RFI-234",
    --   "actualReworkCost": 4200,
    --   "lessonLearned": "Check beam depths before routing supply ducts in corridor"
    -- }
);

CREATE INDEX idx_clashes_test_run ON clashes(clash_test_run_id);
CREATE INDEX idx_clashes_test ON clashes(clash_test_id);
CREATE INDEX idx_clashes_hash ON clashes(clash_hash);
CREATE INDEX idx_clashes_status ON clashes(status);
CREATE INDEX idx_clashes_priority ON clashes(priority);
CREATE INDEX idx_clashes_assigned ON clashes(assigned_to);
CREATE INDEX idx_clashes_storey ON clashes(storey_name);
CREATE INDEX idx_clashes_point ON clashes USING GIST(clash_point);
CREATE INDEX idx_clashes_element_a ON clashes(element_a_id);
CREATE INDEX idx_clashes_element_b ON clashes(element_b_id);
CREATE INDEX idx_clashes_ai ON clashes USING GIN(ai_analysis jsonb_path_ops);

-- Clash group memberships
CREATE TABLE clash_groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    group_type      VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    assigned_to     UUID REFERENCES users(id),
    clash_count     INTEGER NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata schema: {
    --   "rootCause": "Misaligned structural grid at column line C",
    --   "affectedStoreys": ["Level 2", "Level 3"],
    --   "estimatedTotalCost": 25000,
    --   "aiClusterConfidence": 0.89
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE clash_group_members (
    clash_group_id  UUID NOT NULL REFERENCES clash_groups(id) ON DELETE CASCADE,
    clash_id        UUID NOT NULL REFERENCES clashes(id) ON DELETE CASCADE,
    PRIMARY KEY (clash_group_id, clash_id)
);
```

### Issue Management with BCF Compatibility

```sql
-- ============================================================
-- ISSUES (BCF-compatible)
-- Normalized: BCF mandatory fields, status, assignment
-- JSONB: viewpoints, component selections, markup
-- ============================================================

CREATE TABLE issues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,

    -- BCF mandatory fields (normalized)
    bcf_guid        UUID NOT NULL DEFAULT gen_random_uuid(),
    title           VARCHAR(500) NOT NULL,
    topic_type      VARCHAR(100) NOT NULL DEFAULT 'Clash',
    topic_status    VARCHAR(100) NOT NULL DEFAULT 'Open',
    priority        VARCHAR(50) DEFAULT 'Normal',
    creation_author VARCHAR(255) NOT NULL,
    creation_date   TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Workflow fields (normalized for filtering)
    assigned_to     UUID REFERENCES users(id),
    due_date        DATE,
    server_assigned_id VARCHAR(50),

    -- Source tracking
    source_clash_id UUID REFERENCES clashes(id),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- BCF optional and extended fields in JSONB
    bcf_extensions  JSONB NOT NULL DEFAULT '{}',
    -- bcf_extensions schema: {
    --   "description": "Duct penetrates beam at grid C-3, Level 2",
    --   "stage": "Coordination",
    --   "labels": ["MEP", "Structural", "Urgent"],
    --   "referenceLinks": ["https://project-docs.example.com/RFI-234"],
    --   "modifiedAuthor": "j.smith@example.com",
    --   "modifiedDate": "2026-02-10T14:30:00Z",
    --   "relatedTopics": ["uuid1", "uuid2"],
    --   "documentReferences": [
    --     {"guid": "uuid", "description": "Structural drawing S-201", "url": "..."}
    --   ]
    -- }
);

CREATE INDEX idx_issues_project ON issues(project_id);
CREATE INDEX idx_issues_status ON issues(topic_status);
CREATE INDEX idx_issues_assigned ON issues(assigned_to);
CREATE INDEX idx_issues_bcf_guid ON issues(bcf_guid);

-- Viewpoints stored as JSONB (complex, variable structure)
CREATE TABLE issue_viewpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    issue_id        UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
    bcf_guid        UUID NOT NULL DEFAULT gen_random_uuid(),
    index_order     INTEGER NOT NULL DEFAULT 0,
    snapshot_path   VARCHAR(500),

    -- The entire viewpoint definition in JSONB
    -- This maps directly to the BCF visinfo.xsd schema
    viewpoint_data  JSONB NOT NULL,
    -- viewpoint_data schema: {
    --   "camera": {
    --     "type": "perspective",
    --     "viewPoint": {"x": 12.5, "y": 8.4, "z": 9.2},
    --     "direction": {"x": 0.3, "y": -0.5, "z": -0.1},
    --     "upVector": {"x": 0.0, "y": 0.0, "z": 1.0},
    --     "fieldOfView": 60.0,
    --     "aspectRatio": 1.778
    --   },
    --   "components": {
    --     "selection": [
    --       {"ifcGuid": "0YvctVUKr0kugbFTf53O9L", "originatingSystem": "Revit"}
    --     ],
    --     "visibility": {
    --       "defaultVisibility": true,
    --       "exceptions": [
    --         {"ifcGuid": "3cUkl32yn9qRSPvBJNPi0E"}
    --       ]
    --     },
    --     "coloring": [
    --       {"color": "FF0000FF", "components": [{"ifcGuid": "0YvctVUKr0kugbFTf53O9L"}]},
    --       {"color": "00FF00FF", "components": [{"ifcGuid": "2T3mG8r5X4HBd6qHnR6v7w"}]}
    --     ]
    --   },
    --   "clippingPlanes": [
    --     {"location": {"x": 0, "y": 0, "z": 6.0}, "direction": {"x": 0, "y": 0, "z": -1}}
    --   ],
    --   "lines": [
    --     {"start": {"x": 12.0, "y": 8.0, "z": 6.5}, "end": {"x": 13.0, "y": 9.0, "z": 6.5}}
    --   ],
    --   "bitmaps": []
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_issue_viewpoints_issue ON issue_viewpoints(issue_id);

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

CREATE TABLE issue_attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    issue_id        UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
    filename        VARCHAR(255) NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Reporting, Analytics, and Audit

```sql
-- ============================================================
-- REPORTING AND ANALYTICS
-- ============================================================

CREATE TABLE clash_daily_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    clash_test_id   UUID NOT NULL REFERENCES clash_tests(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,

    -- Normalized counts for fast aggregation
    total_clashes   INTEGER NOT NULL DEFAULT 0,
    new_clashes     INTEGER NOT NULL DEFAULT 0,
    active_clashes  INTEGER NOT NULL DEFAULT 0,
    resolved_clashes INTEGER NOT NULL DEFAULT 0,
    ignored_clashes INTEGER NOT NULL DEFAULT 0,
    critical_count  INTEGER NOT NULL DEFAULT 0,
    high_count      INTEGER NOT NULL DEFAULT 0,
    medium_count    INTEGER NOT NULL DEFAULT 0,
    low_count       INTEGER NOT NULL DEFAULT 0,

    -- Extended metrics in JSONB (varies as we add new analytics)
    extended_metrics JSONB NOT NULL DEFAULT '{}',
    -- extended_metrics schema: {
    --   "avgResolutionHours": 48.5,
    --   "p95ResolutionHours": 120.0,
    --   "aiFilteredCount": 234,
    --   "aiFilterRate": 0.32,
    --   "disciplinePairBreakdown": {
    --     "Mechanical-Structural": {"total": 89, "resolved": 45, "critical": 12},
    --     "Electrical-Architectural": {"total": 34, "resolved": 20, "critical": 3}
    --   },
    --   "storeyBreakdown": {
    --     "Level 2": {"total": 67, "resolved": 30},
    --     "Level 3": {"total": 56, "resolved": 25}
    --   },
    --   "teamWorkload": {
    --     "user-uuid-1": {"assigned": 23, "resolved": 8, "overdue": 3},
    --     "user-uuid-2": {"assigned": 15, "resolved": 12, "overdue": 0}
    --   }
    -- }

    UNIQUE(clash_test_id, snapshot_date)
);

CREATE INDEX idx_snapshots_project_date ON clash_daily_snapshots(project_id, snapshot_date);

-- Audit log with structured JSONB for changes
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    project_id      UUID REFERENCES projects(id),
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
    -- changes schema: {
    --   "status": {"from": "new", "to": "active"},
    --   "priority": {"from": "medium", "to": "critical"},
    --   "assignedTo": {"from": null, "to": "user-uuid"}
    -- }
    context         JSONB NOT NULL DEFAULT '{}',
    -- context schema: {
    --   "ipAddress": "192.168.1.100",
    --   "userAgent": "Mozilla/5.0...",
    --   "sessionId": "sess-uuid",
    --   "source": "web_ui"  -- web_ui, api, bcf_import, system
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_project ON audit_log(project_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

-- BCF exchange tracking
CREATE TABLE bcf_exchanges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    direction       VARCHAR(10) NOT NULL,
    bcf_version     VARCHAR(10) NOT NULL,
    filename        VARCHAR(255) NOT NULL,
    topics_count    INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'completed',
    details         JSONB NOT NULL DEFAULT '{}',
    -- details schema: {
    --   "topicsCreated": 12,
    --   "topicsUpdated": 5,
    --   "topicsSkipped": 2,
    --   "errors": ["Topic guid-xyz had invalid viewpoint data"],
    --   "mappingLog": [...]
    -- }
    performed_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### AI/ML Pipeline Support

```sql
-- ============================================================
-- AI/ML PIPELINE
-- Heavy use of JSONB since ML feature vectors and metrics evolve frequently
-- ============================================================

CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_type      VARCHAR(50) NOT NULL,
    version         VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT FALSE,

    -- All model metadata in JSONB (schema evolves with each model type)
    model_metadata  JSONB NOT NULL,
    -- model_metadata schema: {
    --   "framework": "xgboost",
    --   "hyperparameters": {"max_depth": 8, "learning_rate": 0.05, "n_estimators": 500},
    --   "featureImportance": {"distance_mm": 0.23, "ifc_type_pair_encoded": 0.18, ...},
    --   "trainingMetrics": {"accuracy": 0.91, "precision": 0.88, "recall": 0.93, "f1": 0.90, "auc": 0.95},
    --   "validationMetrics": {"accuracy": 0.89, "precision": 0.85, ...},
    --   "trainingSamplesCount": 15000,
    --   "trainingDurationSeconds": 340,
    --   "featureNames": ["distance_mm", "intersection_volume", "ifc_type_a_encoded", ...],
    --   "artifactPath": "s3://ml-models/relevance/v2.3.1/model.xgb",
    --   "datasetVersionHash": "sha256:abc123..."
    -- }

    trained_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    activated_at    TIMESTAMPTZ
);

CREATE TABLE ml_training_samples (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clash_id        UUID NOT NULL REFERENCES clashes(id) ON DELETE CASCADE,

    -- Feature data as JSONB (features change as models evolve)
    features        JSONB NOT NULL,
    -- features schema: {
    --   "distance_mm": -42.3,
    --   "intersection_volume_mm3": 145230.5,
    --   "element_a_ifc_type": "IfcDuctSegment",
    --   "element_b_ifc_type": "IfcBeam",
    --   "element_a_volume_m3": 0.45,
    --   "element_b_volume_m3": 1.2,
    --   "storey_elevation_m": 8.4,
    --   "discipline_pair": "Mechanical-Structural",
    --   "element_a_is_load_bearing": false,
    --   "element_b_is_load_bearing": true,
    --   "nearby_clash_count_5m": 4,
    --   "same_system_clash_count": 2,
    --   "historical_resolution_rate_for_type_pair": 0.78
    -- }

    labels          JSONB NOT NULL,
    -- labels schema: {
    --   "isRelevant": true,
    --   "severity": "high",
    --   "resolutionType": "design_change",
    --   "resolutionTimeHours": 72.5,
    --   "labeledBy": "user-uuid",
    --   "labeledAt": "2026-02-15T09:00:00Z",
    --   "labelSource": "manual"  -- manual, inferred_from_resolution
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ml_training_clash ON ml_training_samples(clash_id);

-- Natural language query interface log
CREATE TABLE nl_query_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    query_text      TEXT NOT NULL,

    query_analysis  JSONB NOT NULL DEFAULT '{}',
    -- query_analysis schema: {
    --   "parsedFilters": {
    --     "disciplines": ["Mechanical", "Structural"],
    --     "storeys": ["Level 3"],
    --     "status": null,
    --     "priority": null,
    --     "ifcTypes": ["IfcDuctSegment"],
    --     "spatial": {"above": "Level 3"}
    --   },
    --   "generatedSQL": "SELECT ... FROM clashes WHERE ...",
    --   "resultCount": 23,
    --   "executionTimeMs": 45,
    --   "confidence": 0.92,
    --   "userFeedback": "helpful"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Example Queries Demonstrating Hybrid Power

### Query JSONB properties alongside normalized columns

```sql
-- Find all fire-rated walls on Level 2 that are involved in unresolved clashes
SELECT
    c.clash_hash,
    c.status,
    c.priority,
    e.ifc_name,
    e.properties->'Pset_WallCommon'->>'FireRating' AS fire_rating,
    e.properties->'Pset_WallCommon'->>'LoadBearing' AS load_bearing,
    c.ai_analysis->>'relevanceScore' AS ai_relevance,
    c.clash_geometry->>'penetrationDepthMm' AS penetration_depth
FROM clashes c
JOIN bim_elements e ON e.id = c.element_a_id
WHERE c.status IN ('new', 'active')
  AND e.ifc_entity_type = 'IfcWall'
  AND e.storey_name = 'Level 2'
  AND e.properties @> '{"Pset_WallCommon": {"FireRating": {"value": "2HR"}}}';

-- Find all clashes where AI predicts high rework cost
SELECT
    c.clash_hash,
    c.element_a_ifc_type,
    c.element_b_ifc_type,
    c.storey_name,
    (c.ai_analysis->>'relevanceScore')::float AS relevance,
    (c.ai_analysis->'predictedReworkCost'->>'max')::int AS max_rework_cost,
    c.ai_analysis->>'suggestedResolution' AS suggestion
FROM clashes c
WHERE c.clash_test_id = $1
  AND c.status = 'new'
  AND (c.ai_analysis->>'relevanceScore')::float > 0.7
ORDER BY (c.ai_analysis->'predictedReworkCost'->>'max')::int DESC
LIMIT 50;

-- Discipline-pair breakdown using JSONB element details
SELECT
    c.element_a_details->>'discipline' AS discipline_a,
    c.element_b_details->>'discipline' AS discipline_b,
    COUNT(*) AS clash_count,
    COUNT(*) FILTER (WHERE c.status = 'resolved') AS resolved_count,
    AVG((c.ai_analysis->>'severityScore')::float) AS avg_severity
FROM clashes c
WHERE c.clash_test_id = $1
GROUP BY 1, 2
ORDER BY clash_count DESC;
```

---

## Pros and Cons

### Pros

1. **Best of both worlds**: Normalized columns provide fast, indexed queries for the fields used in every listing, filter, and sort operation (status, priority, storey, assignment). JSONB columns absorb the enormous variability of IFC properties, AI metadata, and BCF viewpoint structures without schema migrations.

2. **No EAV explosion**: Instead of 25 million rows in an EAV property table (500K elements x 50 properties each), element properties are stored as JSONB on the element row itself. GIN indexes with `jsonb_path_ops` enable efficient containment queries (`@>`) without the join overhead.

3. **BCF round-trip fidelity**: BCF viewpoint data (camera, components, clipping planes, markup lines) maps naturally to JSONB. Import and export become simple JSON-to-XML and XML-to-JSON transformations without any lossy mapping through intermediate relational tables.

4. **Schema evolution without migrations**: When a new IFC version adds properties, or the AI pipeline produces new scoring dimensions, or a new integration adds custom metadata, the JSONB columns absorb these changes with no ALTER TABLE required. Application code handles versioning through defensive JSON parsing.

5. **Single technology stack**: Everything runs on PostgreSQL -- no separate document database, no separate search engine for basic queries. PostGIS provides spatial operations, JSONB provides document flexibility, and standard SQL provides relational integrity. This dramatically simplifies deployment for self-hosted installations.

6. **Partial index opportunities**: PostgreSQL supports partial indexes on JSONB expressions, enabling highly targeted performance optimizations.

```sql
-- Index only high-relevance clashes for dashboard queries
CREATE INDEX idx_clashes_high_relevance ON clashes (clash_test_id)
WHERE (ai_analysis->>'relevanceScore')::float > 0.7;
```

7. **Familiar to developers**: Most backend developers are comfortable with PostgreSQL and JSON. The learning curve is minimal compared to event sourcing, graph databases, or specialized BIM data stores.

### Cons

1. **JSONB query performance limits**: While GIN indexes make containment queries fast (`properties @> '{"Pset_WallCommon": {"FireRating": {"value": "2HR"}}}'`), complex JSONB traversals with type casting (e.g., `(ai_analysis->>'relevanceScore')::float > 0.7`) cannot use GIN indexes and require sequential scans or expression indexes.

2. **No schema enforcement on JSONB**: The database does not validate JSONB structure. A bug in the IFC parser could write malformed property data that is only discovered when the UI tries to render it. JSON Schema validation must be implemented in the application layer.

3. **JSONB size overhead**: JSONB stores field names repeatedly in every row. An element with 50 properties stores the property set names and property names in every row, consuming more storage than a normalized table where column names are stored once in the catalog.

4. **Inconsistent query patterns**: Developers must choose between SQL column access (`WHERE status = 'new'`) and JSONB path access (`WHERE ai_analysis->>'relevanceScore'`) for different fields on the same table. This creates inconsistency in query patterns and can lead to bugs when a field is expected in the wrong location.

5. **JSONB update granularity**: Updating a single property in a JSONB column requires reading the entire JSONB value, modifying it, and writing it back (`jsonb_set()`). For large JSONB objects (e.g., an element with 100 properties), this is less efficient than updating a single column in a normalized table.

6. **Reporting tool compatibility**: Some BI tools (Tableau, Power BI) have limited support for querying JSONB columns. Generating reports may require creating views that extract commonly needed JSONB fields into virtual columns.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Database | PostgreSQL 16+ with PostGIS 3.4+ | Native JSONB + spatial + relational in one engine |
| JSONB validation | ajv (JS) or jsonschema (Python) | Application-layer schema validation before writes |
| JSONB indexing | GIN with jsonb_path_ops | Efficient containment queries on properties |
| Expression indexes | CREATE INDEX ON (...::float) | For frequently queried JSONB numeric fields |
| Connection pooling | PgBouncer | Connection management for web applications |
| Migrations | Alembic (Python) or Prisma Migrate (TypeScript) | Schema versioning for normalized columns |
| Object storage | S3/MinIO | IFC files, geometry meshes, snapshots |
| Caching | Redis | Dashboard aggregations, session data |
| Full-text search | PostgreSQL tsvector (built-in) | Sufficient for element name/description search |
| BI integration | dbt + materialized views | Transform JSONB into flat views for BI tools |

---

## Migration and Scaling Considerations

### Initial Deployment

- Single PostgreSQL 16 instance (8 CPU, 32GB RAM, 500GB NVMe)
- PostGIS extension for spatial queries
- GIN indexes on all JSONB columns
- TOAST compression for large JSONB values (automatic in PostgreSQL)
- Application-level JSON schema validation on write paths

### Growth Phase (100+ projects, millions of elements)

- Read replicas for dashboard and reporting queries
- Table partitioning:

```sql
-- Partition bim_elements by model_version_id for efficient bulk loading/deletion
CREATE TABLE bim_elements (
    -- ... columns as above ...
) PARTITION BY HASH (model_version_id);

-- 32 partitions for good distribution
CREATE TABLE bim_elements_p00 PARTITION OF bim_elements FOR VALUES WITH (MODULUS 32, REMAINDER 0);
CREATE TABLE bim_elements_p01 PARTITION OF bim_elements FOR VALUES WITH (MODULUS 32, REMAINDER 1);
-- ... through p31

-- Partition clashes by clash_test_id
CREATE TABLE clashes (
    -- ... columns as above ...
) PARTITION BY HASH (clash_test_id);
```

- Materialized views for BI tool access:

```sql
-- Flatten JSONB for BI tools
CREATE MATERIALIZED VIEW mv_clash_flat AS
SELECT
    c.id,
    c.clash_hash,
    c.clash_test_id,
    c.status,
    c.priority,
    c.storey_name,
    c.distance_mm,
    c.element_a_ifc_type,
    c.element_b_ifc_type,
    c.element_a_details->>'discipline' AS discipline_a,
    c.element_b_details->>'discipline' AS discipline_b,
    c.element_a_details->>'name' AS element_a_name,
    c.element_b_details->>'name' AS element_b_name,
    (c.ai_analysis->>'relevanceScore')::float AS ai_relevance,
    (c.ai_analysis->>'severityScore')::float AS ai_severity,
    (c.clash_geometry->>'penetrationDepthMm')::float AS penetration_depth,
    (c.clash_geometry->>'intersectionVolumeMm3')::float AS intersection_volume,
    c.first_seen_at,
    c.resolved_at,
    EXTRACT(EPOCH FROM (c.resolved_at - c.first_seen_at))/3600 AS resolution_hours,
    c.resolution->>'type' AS resolution_type,
    (c.resolution->>'actualReworkCost')::int AS actual_rework_cost
FROM clashes c;

-- Refresh on schedule
-- SELECT cron.schedule('refresh_mv_clash_flat', '*/15 * * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_clash_flat');
```

### Enterprise Scale

- Citus extension for horizontal distribution (shard by project_id)
- Dedicated analytics instance with pre-aggregated data
- JSONB column compression using PostgreSQL 16's LZ4 TOAST compression
- Archive old projects: move JSONB properties to cold storage, keep only normalized fields for historical queries
- Consider promoting frequently-queried JSONB fields to normalized columns via migration:

```sql
-- If ai_relevance_score is queried in 80% of queries, promote it
ALTER TABLE clashes ADD COLUMN ai_relevance_score DOUBLE PRECISION
    GENERATED ALWAYS AS ((ai_analysis->>'relevanceScore')::double precision) STORED;
CREATE INDEX idx_clashes_ai_relevance_col ON clashes(ai_relevance_score);
```

### Data Migration from Existing Systems

1. **From Navisworks**: Export clash reports as XML/HTML, parse into clashes table. Map Navisworks element GUIDs to `element_a_global_id`/`element_b_global_id`. Store Navisworks-specific metadata in `custom_metadata` JSONB.

2. **From BIMcollab**: Use BCF export to generate BCF XML files. Parse topic/viewpoint/comment XML into issues table. Store viewpoint data directly in `viewpoint_data` JSONB.

3. **From Solibri**: Export BCF files and import. Solibri-specific rule classifications go into the clash `ai_analysis` JSONB under a `solibriClassification` key.

4. **From spreadsheet-based tracking**: Map columns to normalized fields where possible (status, priority, assignment). Remaining custom columns become entries in `resolution` or `custom_metadata` JSONB.
