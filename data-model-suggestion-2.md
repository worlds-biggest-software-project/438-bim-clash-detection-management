# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) where every state change in the BIM clash detection lifecycle is captured as an immutable domain event. The write side persists events to an append-only event store (PostgreSQL or EventStoreDB), while multiple read-model projections are built asynchronously to serve different query patterns: a relational projection for issue management, a denormalized projection for dashboard analytics, and a search projection for natural-language queries.

## Design Philosophy

BIM clash detection has a naturally event-driven lifecycle. Models are uploaded, elements are extracted, clash tests are configured, clashes are detected, reviewed, assigned, commented on, and resolved. Every one of these state transitions has business significance: audit trails are critical for contractual disputes, understanding who changed a clash status and when can have legal implications in construction claims, and the ability to replay history enables "what-if" analysis. Event sourcing makes this history a first-class citizen rather than an afterthought bolted onto a CRUD model.

The CQRS split is justified by the asymmetric read/write workload: clash detection produces bursts of writes (thousands of clashes detected in a single run), while coordination teams perform heavy reads (filtering, grouping, trend analysis, dashboard views) throughout the project lifecycle.

---

## Event Store Schema

### Core Event Store (PostgreSQL)

```sql
-- ============================================================
-- EVENT STORE FOUNDATION
-- ============================================================

-- Global event log: the single source of truth
CREATE TABLE event_store (
    global_position     BIGSERIAL PRIMARY KEY,  -- monotonically increasing global order
    stream_id           VARCHAR(500) NOT NULL,   -- aggregate identifier e.g. "project:abc123"
    stream_position     INTEGER NOT NULL,        -- position within the stream (optimistic concurrency)
    event_type          VARCHAR(200) NOT NULL,    -- e.g. "ClashDetected", "ClashStatusChanged"
    event_data          JSONB NOT NULL,           -- the event payload
    metadata            JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, user_id, timestamp
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(stream_id, stream_position)
);

-- Indexes for projection building and subscription
CREATE INDEX idx_event_store_stream ON event_store(stream_id, stream_position);
CREATE INDEX idx_event_store_type ON event_store(event_type);
CREATE INDEX idx_event_store_created ON event_store(created_at);
CREATE INDEX idx_event_store_global ON event_store(global_position);

-- Category index for subscribing to all events of a type prefix
-- e.g., all "clash_test:" streams
CREATE INDEX idx_event_store_stream_prefix ON event_store(split_part(stream_id, ':', 1));

-- Snapshot store for aggregate rehydration optimization
CREATE TABLE event_snapshots (
    stream_id           VARCHAR(500) NOT NULL,
    stream_position     INTEGER NOT NULL,       -- position at which snapshot was taken
    aggregate_type      VARCHAR(100) NOT NULL,
    state_data          JSONB NOT NULL,          -- serialized aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY(stream_id, stream_position)
);

-- Projection checkpoint tracking
CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(200) PRIMARY KEY,
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status              VARCHAR(50) NOT NULL DEFAULT 'running', -- running, paused, rebuilding, failed
    error_message       TEXT
);

-- Dead letter queue for events that failed projection processing
CREATE TABLE dead_letter_events (
    id                  BIGSERIAL PRIMARY KEY,
    global_position     BIGINT NOT NULL,
    projection_name     VARCHAR(200) NOT NULL,
    event_type          VARCHAR(200) NOT NULL,
    event_data          JSONB NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Aggregate Streams and Event Types

### Stream Naming Convention

Streams follow the pattern `{aggregate_type}:{aggregate_id}`, enabling category-based subscriptions.

| Stream Pattern | Example | Description |
|---|---|---|
| `organization:{id}` | `organization:550e8400-...` | Organization lifecycle |
| `project:{id}` | `project:6ba7b810-...` | Project lifecycle |
| `model:{id}` | `model:7c9e6679-...` | Model file lifecycle |
| `model_version:{id}` | `model_version:8f14e45f-...` | Model version processing |
| `clash_test:{id}` | `clash_test:1b9d6bcd-...` | Clash test configuration and runs |
| `clash:{hash}` | `clash:a1b2c3d4e5f6...` | Individual clash lifecycle (keyed by stable hash) |
| `issue:{id}` | `issue:3c9909af-...` | BCF-compatible issue lifecycle |
| `ml_pipeline:{id}` | `ml_pipeline:58e0a7d7-...` | ML model training and deployment |

### Complete Event Type Catalog

```typescript
// ============================================================
// ORGANIZATION EVENTS
// ============================================================

interface OrganizationCreated {
    type: "OrganizationCreated";
    data: {
        organizationId: string;
        name: string;
        slug: string;
        subscriptionTier: string;
        createdBy: string;
    };
}

interface MemberInvited {
    type: "MemberInvited";
    data: {
        organizationId: string;
        userId: string;
        email: string;
        role: "owner" | "admin" | "member" | "viewer";
        invitedBy: string;
    };
}

interface MemberRoleChanged {
    type: "MemberRoleChanged";
    data: {
        organizationId: string;
        userId: string;
        previousRole: string;
        newRole: string;
        changedBy: string;
    };
}

// ============================================================
// PROJECT EVENTS
// ============================================================

interface ProjectCreated {
    type: "ProjectCreated";
    data: {
        projectId: string;
        organizationId: string;
        name: string;
        code: string;
        description: string;
        projectType: string;
        units: "meters" | "feet" | "millimeters";
        coordinateSystem: string;
        createdBy: string;
    };
}

interface DisciplineAdded {
    type: "DisciplineAdded";
    data: {
        projectId: string;
        disciplineId: string;
        name: string;
        abbreviation: string;
        color: string;
    };
}

interface ProjectMemberAdded {
    type: "ProjectMemberAdded";
    data: {
        projectId: string;
        userId: string;
        role: string;
        discipline: string;
        addedBy: string;
    };
}

// ============================================================
// MODEL FEDERATION EVENTS
// ============================================================

interface ModelFileUploaded {
    type: "ModelFileUploaded";
    data: {
        modelFileId: string;
        projectId: string;
        disciplineId: string;
        name: string;
        sourceApplication: string;
        sourceFormat: string;
        fileSizeBytes: number;
        storagePath: string;
        checksumSha256: string;
        uploadedBy: string;
    };
}

interface ModelVersionCreated {
    type: "ModelVersionCreated";
    data: {
        modelVersionId: string;
        modelFileId: string;
        versionNumber: number;
        fileSizeBytes: number;
        storagePath: string;
        checksumSha256: string;
        uploadedBy: string;
    };
}

interface ModelVersionProcessingStarted {
    type: "ModelVersionProcessingStarted";
    data: {
        modelVersionId: string;
        processingJobId: string;
        startedAt: string; // ISO 8601
    };
}

interface ElementsExtracted {
    type: "ElementsExtracted";
    data: {
        modelVersionId: string;
        elementCount: number;
        elementTypeCounts: Record<string, number>; // {"IfcWall": 342, "IfcBeam": 128, ...}
        extractionDurationMs: number;
    };
}

interface ElementExtracted {
    type: "ElementExtracted";
    data: {
        elementId: string;
        modelVersionId: string;
        ifcGlobalId: string;
        ifcEntityType: string;
        ifcName: string;
        buildingName: string;
        storeyName: string;
        classificationCode: string;
        bboxMin: { x: number; y: number; z: number };
        bboxMax: { x: number; y: number; z: number };
        centroid: { x: number; y: number; z: number };
        volumeM3: number;
        properties: Record<string, Record<string, string>>; // {psetName: {propName: value}}
    };
}

interface ModelVersionProcessingCompleted {
    type: "ModelVersionProcessingCompleted";
    data: {
        modelVersionId: string;
        elementCount: number;
        ifcProjectGuid: string;
        ifcProjectName: string;
        durationMs: number;
    };
}

interface ModelVersionProcessingFailed {
    type: "ModelVersionProcessingFailed";
    data: {
        modelVersionId: string;
        errorMessage: string;
        errorStack: string;
    };
}

interface FederatedModelCreated {
    type: "FederatedModelCreated";
    data: {
        federatedModelId: string;
        projectId: string;
        name: string;
        modelVersionIds: string[];
        createdBy: string;
    };
}

interface FederatedModelMemberAdded {
    type: "FederatedModelMemberAdded";
    data: {
        federatedModelId: string;
        modelVersionId: string;
        transformMatrix: number[]; // 16 floats
    };
}

// ============================================================
// CLASH TEST CONFIGURATION EVENTS
// ============================================================

interface ClashTestCreated {
    type: "ClashTestCreated";
    data: {
        clashTestId: string;
        projectId: string;
        federatedModelId: string;
        name: string;
        testType: "hard" | "soft" | "clearance" | "workflow_4d";
        setA: {
            name: string;
            disciplineId: string;
            modelVersionId: string;
            ifcTypes: string[];
            storeyFilter: string[];
        };
        setB: {
            name: string;
            disciplineId: string;
            modelVersionId: string;
            ifcTypes: string[];
            storeyFilter: string[];
        };
        toleranceMm: number;
        clearanceMm: number;
        createdBy: string;
    };
}

interface ClashTestConfigurationUpdated {
    type: "ClashTestConfigurationUpdated";
    data: {
        clashTestId: string;
        changes: Record<string, { from: any; to: any }>;
        updatedBy: string;
    };
}

interface ClashIgnoreRuleAdded {
    type: "ClashIgnoreRuleAdded";
    data: {
        ruleId: string;
        clashTestId: string;
        ruleType: string;
        description: string;
        criteria: Record<string, any>;
        createdBy: string;
    };
}

// ============================================================
// CLASH DETECTION EXECUTION EVENTS
// ============================================================

interface ClashTestRunStarted {
    type: "ClashTestRunStarted";
    data: {
        clashTestRunId: string;
        clashTestId: string;
        runNumber: number;
        triggeredBy: "manual" | "model_upload" | "scheduled";
        triggeredByUser: string;
        federatedModelSnapshot: Array<{
            modelVersionId: string;
            modelName: string;
            versionNumber: number;
        }>;
        startedAt: string;
    };
}

interface ClashDetected {
    type: "ClashDetected";
    data: {
        clashHash: string; // stable identifier across runs
        clashTestRunId: string;
        clashTestId: string;
        clashNumber: number;
        elementA: {
            id: string;
            ifcGlobalId: string;
            ifcType: string;
            name: string;
            modelVersionId: string;
        };
        elementB: {
            id: string;
            ifcGlobalId: string;
            ifcType: string;
            name: string;
            modelVersionId: string;
        };
        clashPoint: { x: number; y: number; z: number };
        distanceMm: number;
        intersectionVolumeMm3: number;
        storeyName: string;
        zoneName: string;
        gridReference: string;
    };
}

interface ClashDisappeared {
    type: "ClashDisappeared";
    data: {
        clashHash: string;
        clashTestRunId: string;
        clashTestId: string;
        previousRunId: string;
        reason: "element_removed" | "element_moved" | "model_updated";
    };
}

interface ClashTestRunCompleted {
    type: "ClashTestRunCompleted";
    data: {
        clashTestRunId: string;
        clashTestId: string;
        totalClashes: number;
        newClashes: number;
        activeClashes: number;
        resolvedClashes: number;
        clashesAppeared: number;
        clashesDisappeared: number;
        elementsCheckedA: number;
        elementsCheckedB: number;
        pairsEvaluated: number;
        durationMs: number;
        completedAt: string;
    };
}

interface ClashTestRunFailed {
    type: "ClashTestRunFailed";
    data: {
        clashTestRunId: string;
        clashTestId: string;
        errorMessage: string;
        failedAt: string;
    };
}

// ============================================================
// CLASH LIFECYCLE EVENTS
// ============================================================

interface ClashStatusChanged {
    type: "ClashStatusChanged";
    data: {
        clashHash: string;
        clashTestId: string;
        previousStatus: string;
        newStatus: "new" | "active" | "reviewed" | "approved" | "resolved" | "ignored";
        changedBy: string;
        reason: string;
    };
}

interface ClashPriorityChanged {
    type: "ClashPriorityChanged";
    data: {
        clashHash: string;
        clashTestId: string;
        previousPriority: string;
        newPriority: "critical" | "high" | "medium" | "low" | "none";
        changedBy: string;
    };
}

interface ClashAssigned {
    type: "ClashAssigned";
    data: {
        clashHash: string;
        clashTestId: string;
        previousAssignee: string | null;
        newAssignee: string;
        assignedBy: string;
    };
}

interface ClashResolved {
    type: "ClashResolved";
    data: {
        clashHash: string;
        clashTestId: string;
        resolutionType: "design_change" | "tolerance_accepted" | "false_positive" | "deferred";
        resolutionNotes: string;
        resolvedBy: string;
    };
}

interface ClashGroupCreated {
    type: "ClashGroupCreated";
    data: {
        groupId: string;
        clashTestId: string;
        name: string;
        groupType: "zone" | "level" | "discipline_pair" | "root_cause" | "ai_cluster";
        clashHashes: string[];
        createdBy: string;
    };
}

// ============================================================
// AI SCORING EVENTS
// ============================================================

interface ClashAIScored {
    type: "ClashAIScored";
    data: {
        clashHash: string;
        clashTestRunId: string;
        relevanceScore: number;
        severityScore: number;
        clusterId: string;
        suggestedResolution: string;
        modelVersion: string;
        scoredAt: string;
    };
}

interface AIModelTrained {
    type: "AIModelTrained";
    data: {
        modelId: string;
        modelType: "relevance_classifier" | "severity_predictor" | "cluster_model";
        version: string;
        trainingSamplesCount: number;
        metrics: {
            accuracy: number;
            precision: number;
            recall: number;
            f1: number;
        };
        artifactPath: string;
    };
}

interface AIModelActivated {
    type: "AIModelActivated";
    data: {
        modelId: string;
        modelType: string;
        previousActiveModelId: string | null;
        activatedBy: string;
    };
}

// ============================================================
// ISSUE / BCF EVENTS
// ============================================================

interface IssueCreated {
    type: "IssueCreated";
    data: {
        issueId: string;
        projectId: string;
        bcfGuid: string;
        title: string;
        description: string;
        topicType: string;
        topicStatus: string;
        priority: string;
        assignedTo: string;
        sourceClashHash: string | null;
        sourceClashTestId: string | null;
        creationAuthor: string;
        serverAssignedId: string;
    };
}

interface IssueStatusChanged {
    type: "IssueStatusChanged";
    data: {
        issueId: string;
        previousStatus: string;
        newStatus: string;
        changedBy: string;
        reason: string;
    };
}

interface IssueCommentAdded {
    type: "IssueCommentAdded";
    data: {
        commentId: string;
        issueId: string;
        bcfGuid: string;
        author: string;
        authorUserId: string;
        commentText: string;
        viewpointId: string | null;
    };
}

interface IssueViewpointAdded {
    type: "IssueViewpointAdded";
    data: {
        viewpointId: string;
        issueId: string;
        bcfGuid: string;
        cameraType: "perspective" | "orthogonal";
        cameraViewPoint: { x: number; y: number; z: number };
        cameraDirection: { x: number; y: number; z: number };
        cameraUpVector: { x: number; y: number; z: number };
        fieldOfView: number;
        selectedComponents: string[];
        exceptionComponents: string[];
        snapshotPath: string;
    };
}

interface BCFImported {
    type: "BCFImported";
    data: {
        exchangeId: string;
        projectId: string;
        bcfVersion: string;
        filename: string;
        topicsImported: number;
        importedBy: string;
    };
}

interface BCFExported {
    type: "BCFExported";
    data: {
        exchangeId: string;
        projectId: string;
        bcfVersion: string;
        filename: string;
        topicsExported: number;
        exportedBy: string;
    };
}
```

---

## Read Model Projections

### Projection 1: Operational Read Model (PostgreSQL)

This projection serves the primary application UI: browsing projects, viewing clash results, managing issues.

```sql
-- ============================================================
-- OPERATIONAL READ MODEL (projected from events)
-- ============================================================

CREATE TABLE rm_projects (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    status          VARCHAR(50) NOT NULL,
    project_type    VARCHAR(100),
    units           VARCHAR(20) NOT NULL,
    model_count     INTEGER NOT NULL DEFAULT 0,
    active_clash_count INTEGER NOT NULL DEFAULT 0,
    open_issue_count INTEGER NOT NULL DEFAULT 0,
    last_activity_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_model_files (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    discipline_name VARCHAR(100) NOT NULL,
    discipline_abbreviation VARCHAR(10) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    source_application VARCHAR(100),
    current_version_number INTEGER NOT NULL DEFAULT 0,
    current_version_id UUID,
    element_count   INTEGER NOT NULL DEFAULT 0,
    processing_status VARCHAR(50) NOT NULL,
    last_uploaded_at TIMESTAMPTZ,
    uploaded_by_name VARCHAR(255)
);

CREATE INDEX idx_rm_model_files_project ON rm_model_files(project_id);

CREATE TABLE rm_clash_tests (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    federated_model_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    test_type       VARCHAR(20) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    set_a_description VARCHAR(500),
    set_b_description VARCHAR(500),
    tolerance_mm    DOUBLE PRECISION,
    total_clashes   INTEGER DEFAULT 0,
    new_clashes     INTEGER DEFAULT 0,
    active_clashes  INTEGER DEFAULT 0,
    resolved_clashes INTEGER DEFAULT 0,
    last_run_at     TIMESTAMPTZ,
    last_run_number INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_clash_tests_project ON rm_clash_tests(project_id);

-- Denormalized clash view for fast list/filter/sort
CREATE TABLE rm_clashes (
    clash_hash      VARCHAR(64) PRIMARY KEY,
    clash_test_id   UUID NOT NULL,
    clash_test_name VARCHAR(255) NOT NULL,
    project_id      UUID NOT NULL,

    -- Current state (updated by status change events)
    status          VARCHAR(50) NOT NULL DEFAULT 'new',
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    assigned_to_id  UUID,
    assigned_to_name VARCHAR(255),
    due_date        DATE,
    resolution_type VARCHAR(50),

    -- Element details (denormalized for display without joins)
    element_a_global_id VARCHAR(64) NOT NULL,
    element_a_ifc_type VARCHAR(100) NOT NULL,
    element_a_name  VARCHAR(500),
    element_a_discipline VARCHAR(100),
    element_b_global_id VARCHAR(64) NOT NULL,
    element_b_ifc_type VARCHAR(100) NOT NULL,
    element_b_name  VARCHAR(500),
    element_b_discipline VARCHAR(100),

    -- Spatial context
    clash_point_x   DOUBLE PRECISION,
    clash_point_y   DOUBLE PRECISION,
    clash_point_z   DOUBLE PRECISION,
    distance_mm     DOUBLE PRECISION,
    storey_name     VARCHAR(255),
    zone_name       VARCHAR(255),
    grid_reference  VARCHAR(100),

    -- AI scores
    ai_relevance_score DOUBLE PRECISION,
    ai_severity_score DOUBLE PRECISION,
    ai_cluster_id   UUID,
    ai_suggested_resolution TEXT,

    -- Lifecycle tracking
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_run_id UUID,
    resolved_at     TIMESTAMPTZ,
    resolved_by_name VARCHAR(255),
    status_change_count INTEGER NOT NULL DEFAULT 0,
    comment_count   INTEGER NOT NULL DEFAULT 0,

    -- Group membership
    group_ids       UUID[],
    group_names     TEXT[],

    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_clashes_test ON rm_clashes(clash_test_id);
CREATE INDEX idx_rm_clashes_project ON rm_clashes(project_id);
CREATE INDEX idx_rm_clashes_status ON rm_clashes(status);
CREATE INDEX idx_rm_clashes_priority ON rm_clashes(priority);
CREATE INDEX idx_rm_clashes_assigned ON rm_clashes(assigned_to_id);
CREATE INDEX idx_rm_clashes_storey ON rm_clashes(storey_name);
CREATE INDEX idx_rm_clashes_relevance ON rm_clashes(ai_relevance_score DESC NULLS LAST);

CREATE TABLE rm_issues (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    bcf_guid        UUID NOT NULL,
    server_assigned_id VARCHAR(50),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    topic_type      VARCHAR(100) NOT NULL,
    topic_status    VARCHAR(100) NOT NULL,
    priority        VARCHAR(50),
    assigned_to_id  UUID,
    assigned_to_name VARCHAR(255),
    due_date        DATE,
    source_clash_hash VARCHAR(64),
    creation_author VARCHAR(255),
    comment_count   INTEGER NOT NULL DEFAULT 0,
    viewpoint_count INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_issues_project ON rm_issues(project_id);
CREATE INDEX idx_rm_issues_status ON rm_issues(topic_status);
CREATE INDEX idx_rm_issues_assigned ON rm_issues(assigned_to_id);

-- Clash history timeline (one row per event, for detail views)
CREATE TABLE rm_clash_history (
    id              BIGSERIAL PRIMARY KEY,
    clash_hash      VARCHAR(64) NOT NULL,
    event_type      VARCHAR(200) NOT NULL,
    event_data      JSONB NOT NULL,
    actor_name      VARCHAR(255),
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_clash_history_hash ON rm_clash_history(clash_hash, occurred_at);
```

### Projection 2: Analytics Read Model (TimescaleDB or ClickHouse)

Optimized for time-series dashboards and trend analysis.

```sql
-- ============================================================
-- ANALYTICS READ MODEL (time-series optimized)
-- ============================================================

-- Using TimescaleDB hypertable for time-series queries
CREATE TABLE analytics_clash_events (
    event_time      TIMESTAMPTZ NOT NULL,
    project_id      UUID NOT NULL,
    clash_test_id   UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL, -- detected, status_changed, resolved, assigned
    clash_hash      VARCHAR(64) NOT NULL,
    status_from     VARCHAR(50),
    status_to       VARCHAR(50),
    priority        VARCHAR(20),
    discipline_a    VARCHAR(100),
    discipline_b    VARCHAR(100),
    storey_name     VARCHAR(255),
    assigned_to_id  UUID,
    ai_relevance    DOUBLE PRECISION,
    ai_severity     DOUBLE PRECISION,
    resolution_type VARCHAR(50),
    resolution_time_hours DOUBLE PRECISION
);

-- Convert to hypertable (TimescaleDB)
-- SELECT create_hypertable('analytics_clash_events', 'event_time');

CREATE INDEX idx_analytics_events_project ON analytics_clash_events(project_id, event_time DESC);
CREATE INDEX idx_analytics_events_test ON analytics_clash_events(clash_test_id, event_time DESC);

-- Pre-aggregated daily metrics (materialized by projection)
CREATE TABLE analytics_daily_metrics (
    metric_date     DATE NOT NULL,
    project_id      UUID NOT NULL,
    clash_test_id   UUID NOT NULL,
    discipline_pair VARCHAR(200), -- "Mechanical-Structural"

    total_open      INTEGER NOT NULL DEFAULT 0,
    total_new       INTEGER NOT NULL DEFAULT 0,
    total_resolved  INTEGER NOT NULL DEFAULT 0,
    total_ignored   INTEGER NOT NULL DEFAULT 0,

    critical_open   INTEGER NOT NULL DEFAULT 0,
    high_open       INTEGER NOT NULL DEFAULT 0,
    medium_open     INTEGER NOT NULL DEFAULT 0,
    low_open        INTEGER NOT NULL DEFAULT 0,

    avg_resolution_hours DOUBLE PRECISION,
    p95_resolution_hours DOUBLE PRECISION,
    ai_filter_rate  DOUBLE PRECISION, -- percentage filtered by AI as low-relevance

    PRIMARY KEY (metric_date, project_id, clash_test_id, discipline_pair)
);

-- Team workload analytics
CREATE TABLE analytics_team_workload (
    metric_date     DATE NOT NULL,
    project_id      UUID NOT NULL,
    user_id         UUID NOT NULL,
    user_name       VARCHAR(255) NOT NULL,
    assigned_count  INTEGER NOT NULL DEFAULT 0,
    resolved_count  INTEGER NOT NULL DEFAULT 0,
    reviewed_count  INTEGER NOT NULL DEFAULT 0,
    avg_resolution_hours DOUBLE PRECISION,
    overdue_count   INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (metric_date, project_id, user_id)
);
```

### Projection 3: Search Read Model (Elasticsearch/OpenSearch)

Optimized for natural-language queries and full-text filtering.

```json
// ============================================================
// ELASTICSEARCH INDEX MAPPING: clashes
// ============================================================
{
  "mappings": {
    "properties": {
      "clash_hash":           { "type": "keyword" },
      "clash_test_id":        { "type": "keyword" },
      "clash_test_name":      { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "project_id":           { "type": "keyword" },
      "project_name":         { "type": "text", "fields": { "keyword": { "type": "keyword" } } },

      "status":               { "type": "keyword" },
      "priority":             { "type": "keyword" },
      "assigned_to_name":     { "type": "text", "fields": { "keyword": { "type": "keyword" } } },

      "element_a": {
        "properties": {
          "ifc_global_id":    { "type": "keyword" },
          "ifc_type":         { "type": "keyword" },
          "name":             { "type": "text" },
          "discipline":       { "type": "keyword" },
          "material":         { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
          "classification":   { "type": "keyword" }
        }
      },
      "element_b": {
        "properties": {
          "ifc_global_id":    { "type": "keyword" },
          "ifc_type":         { "type": "keyword" },
          "name":             { "type": "text" },
          "discipline":       { "type": "keyword" },
          "material":         { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
          "classification":   { "type": "keyword" }
        }
      },

      "clash_point":          { "type": "geo_point" },
      "clash_point_3d":       { "type": "float", "index": false },
      "distance_mm":          { "type": "float" },
      "storey_name":          { "type": "keyword" },
      "zone_name":            { "type": "keyword" },
      "grid_reference":       { "type": "keyword" },

      "ai_relevance_score":   { "type": "float" },
      "ai_severity_score":    { "type": "float" },
      "ai_suggested_resolution": { "type": "text" },

      "first_seen_at":        { "type": "date" },
      "resolved_at":          { "type": "date" },
      "updated_at":           { "type": "date" },

      "description":          { "type": "text" },
      "comments":             { "type": "text" },
      "labels":               { "type": "keyword" },

      "all_text": {
        "type": "text",
        "analyzer": "standard"
      }
    }
  }
}
```

---

## Event Processing Pipeline

### Projection Handler Architecture

```
                     ┌──────────────────┐
   Commands ────────>│   Command Handler │
                     │   (Validates,     │
                     │    Appends Events)│
                     └────────┬─────────┘
                              │ Events
                              v
                     ┌──────────────────┐
                     │   Event Store     │
                     │   (PostgreSQL)    │
                     └────────┬─────────┘
                              │ Subscription
                    ┌─────────┼─────────────┐
                    v         v             v
           ┌─────────────┐ ┌──────────┐ ┌──────────────┐
           │ Operational  │ │Analytics │ │   Search     │
           │ Projection   │ │Projection│ │  Projection  │
           │ (PostgreSQL) │ │(Timescale│ │(Elasticsearch│
           │              │ │   DB)    │ │              │
           └─────────────┘ └──────────┘ └──────────────┘
                    │         │             │
                    v         v             v
           ┌─────────────┐ ┌──────────┐ ┌──────────────┐
           │  App Queries │ │Dashboard │ │  NL Query    │
           │  (REST API)  │ │  & BI    │ │  Interface   │
           └─────────────┘ └──────────┘ └──────────────┘
```

### Example Projection Handler (Pseudocode)

```python
class ClashProjectionHandler:
    """Processes clash-related events into the operational read model."""

    async def handle(self, event: Event) -> None:
        match event.event_type:
            case "ClashDetected":
                await self._handle_clash_detected(event)
            case "ClashStatusChanged":
                await self._handle_status_changed(event)
            case "ClashAIScored":
                await self._handle_ai_scored(event)
            case "ClashResolved":
                await self._handle_resolved(event)
            case "ClashGroupCreated":
                await self._handle_group_created(event)

    async def _handle_clash_detected(self, event: Event) -> None:
        data = event.event_data
        # Upsert into rm_clashes (clash may reappear in a new run)
        await self.db.execute("""
            INSERT INTO rm_clashes (
                clash_hash, clash_test_id, clash_test_name, project_id,
                status, priority,
                element_a_global_id, element_a_ifc_type, element_a_name,
                element_b_global_id, element_b_ifc_type, element_b_name,
                clash_point_x, clash_point_y, clash_point_z,
                distance_mm, storey_name, zone_name, grid_reference,
                first_seen_at, last_seen_run_id, updated_at
            ) VALUES ($1, $2, ..., $n)
            ON CONFLICT (clash_hash) DO UPDATE SET
                last_seen_run_id = EXCLUDED.last_seen_run_id,
                updated_at = EXCLUDED.updated_at
        """, ...)

        # Also insert into history timeline
        await self.db.execute("""
            INSERT INTO rm_clash_history (clash_hash, event_type, event_data, occurred_at)
            VALUES ($1, $2, $3, $4)
        """, data["clashHash"], "ClashDetected", event.event_data, event.metadata["timestamp"])

    async def _handle_status_changed(self, event: Event) -> None:
        data = event.event_data
        await self.db.execute("""
            UPDATE rm_clashes SET
                status = $1,
                status_change_count = status_change_count + 1,
                updated_at = $2
            WHERE clash_hash = $3
        """, data["newStatus"], event.metadata["timestamp"], data["clashHash"])

        # Update aggregate counts on the clash test
        await self._recalculate_test_counts(data["clashTestId"])

        # Insert history record
        await self.db.execute("""
            INSERT INTO rm_clash_history (clash_hash, event_type, event_data, actor_name, occurred_at)
            VALUES ($1, $2, $3, $4, $5)
        """, data["clashHash"], "ClashStatusChanged", event.event_data,
             data["changedBy"], event.metadata["timestamp"])
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design**: Every state change is an immutable event. For construction projects where clash resolution decisions may be scrutinized during disputes or claims, the event log provides an indisputable record of who changed what and when, without any separate audit mechanism.

2. **Temporal queries are trivial**: "What was the state of this clash on March 15?" is answered by replaying events up to that date. This is critical for projects spanning years where historical state matters for compliance reporting and post-mortem analysis.

3. **Multiple optimized read models**: The CQRS split allows each query pattern to have its own optimized projection: a denormalized relational model for the UI, a time-series store for dashboards, and a search engine for natural-language queries. Each projection is independently tunable.

4. **Natural fit for clash detection bursts**: A single clash test run may produce 10,000+ ClashDetected events. The event store efficiently handles these write bursts, while read model projections update asynchronously, preventing detection runs from blocking the UI.

5. **Replay and rebuild capability**: If a bug in a projection handler causes incorrect read model state, the projection can be torn down and rebuilt by replaying all events from the beginning. This is safer than trying to write migration scripts to fix corrupted CRUD state.

6. **Event-driven integrations**: External system integrations (BCF export, Procore sync, webhook notifications, Slack alerts) subscribe to events rather than polling. When a ClashStatusChanged event occurs, all interested consumers are notified without coupling to the write model.

7. **AI training pipeline alignment**: ML model training inherently needs historical data. The event store provides a natural training corpus: every clash detection result, every resolution decision, every human override of an AI score is captured as an event.

### Cons

1. **Significant implementation complexity**: Event sourcing requires building projection handlers, managing checkpoint tracking, handling idempotency, implementing dead-letter queues, and designing compensating events. The development effort is substantially higher than a CRUD approach.

2. **Eventual consistency challenges**: Read models lag behind the event store. After a user resolves a clash, the UI may still show it as active for a brief period. For BIM coordination meetings where teams are making real-time decisions, this lag (even if sub-second) creates confusion.

3. **Event schema evolution difficulty**: Once events are persisted, changing their structure requires versioning strategies (upcasting, event versioning). If the ClashDetected event schema needs a new field added mid-project, both old and new events must be handled correctly.

4. **Debugging complexity**: When a read model shows incorrect data, the developer must trace through event replay logic rather than simply querying the current state. This requires specialized tooling and expertise that most BIM/AEC development teams do not have.

5. **Storage growth**: The event store grows unboundedly. A large project generating 50,000 clashes across 100 runs will produce millions of events. Snapshot strategies mitigate aggregate rehydration cost but do not reduce storage requirements.

6. **Overkill for simple queries**: Basic CRUD queries (list all projects, get model details) still require maintaining a separate read model. The event sourcing overhead provides no benefit for entities that rarely change state, like organizations, users, and project configurations.

7. **Team skill requirements**: Event sourcing and CQRS demand expertise that is uncommon in the AEC software development community. Hiring and training costs are real constraints for an open-source project seeking contributors.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Event Store | PostgreSQL with LISTEN/NOTIFY or EventStoreDB | PostgreSQL keeps the stack simpler; EventStoreDB offers native projections and subscriptions |
| Operational Read Model | PostgreSQL | Proven, same technology as event store, rich query support |
| Analytics Read Model | TimescaleDB (PostgreSQL extension) | Native time-series hypertables, continuous aggregates |
| Search Read Model | OpenSearch or Elasticsearch | Full-text search, aggregation, natural-language query support |
| Event Bus | NATS JetStream or Apache Kafka | Durable pub/sub for projection handlers and integrations |
| Command Handlers | Node.js/TypeScript or Python | Strong event sourcing libraries (Eventuous, esdbclient) |
| Projection Framework | Custom or Eventuous/.NET | Checkpoint management, idempotency, error handling |
| Saga/Process Managers | Temporal.io | Long-running processes like "detect clashes, score with AI, notify team" |

---

## Migration and Scaling Considerations

### Initial Deployment

- Single PostgreSQL instance serves as both event store and operational read model
- Event processing is in-process (same application, no external message bus)
- OpenSearch runs as a single node for search projection
- Snapshots taken every 100 events per aggregate stream

### Growth Phase

- Separate PostgreSQL instances for event store (write-optimized) and read models (read-optimized)
- Introduce NATS JetStream or Kafka for durable event distribution to projections
- TimescaleDB instance for analytics projection with continuous aggregates
- Event store partitioned by month for archival

```sql
-- Partition event store by time for manageable archival
CREATE TABLE event_store (
    -- ... columns as above ...
) PARTITION BY RANGE (created_at);

CREATE TABLE event_store_2026_q1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
-- ...
```

### Enterprise Scale

- EventStoreDB cluster (3+ nodes) replaces PostgreSQL event store for native subscription support
- Multiple projection worker instances with partition-based work distribution
- Archive old event partitions to cold storage (S3 + Parquet)
- Maintain "hot" window of last 6 months in the active event store
- Cross-region replication for global teams

### Migration Strategy from CRUD

For organizations migrating from an existing CRUD system:

1. **Dual-write phase**: New writes go to both the legacy CRUD database and the event store
2. **Backfill historical data**: Generate synthetic events from existing CRUD records (e.g., create a "ClashImported" event for each existing clash record with its current state)
3. **Build projections**: Replay all events to populate read models
4. **Cut over reads**: Switch queries to read from projections
5. **Cut over writes**: Switch commands to only write events
6. **Decommission legacy**: Remove dual-write after validation period

### Event Store Compaction (Long-running Projects)

For projects spanning 3+ years, event streams for individual clashes can grow long. Compaction strategies:

```python
# Snapshot strategy: after every 50 events on a clash stream,
# create a snapshot of current state
async def maybe_snapshot(stream_id: str, position: int, state: ClashState):
    if position % 50 == 0:
        await event_store.save_snapshot(
            stream_id=stream_id,
            position=position,
            state=state.to_dict()
        )

# Archival strategy: for completed projects, collapse event streams
# into a single "ProjectArchived" snapshot event
async def archive_project(project_id: str):
    final_state = await rebuild_project_state(project_id)
    await event_store.append(
        stream_id=f"project:{project_id}",
        event_type="ProjectArchived",
        data={"finalState": final_state, "archivedAt": datetime.utcnow().isoformat()}
    )
    # Move detailed events to cold storage
    await archive_events_to_s3(stream_prefix=f"project:{project_id}")
```
