# Data Model Suggestion 4: Graph Database (Neo4j) + Relational Sidecar

## Overview

A domain-specific approach using Neo4j as the primary store for BIM element relationships, spatial proximity, clash detection results, and resolution pathways, combined with a PostgreSQL sidecar for transactional workflow data (user management, project configuration, issue lifecycle, BCF exchange). The graph model naturally represents the interconnected nature of BIM data: elements are connected to spaces, spaces to storeys, storeys to buildings, clashes connect pairs of elements, and resolution patterns form paths through historical data. This structure enables queries that are awkward or expensive in relational databases but trivial in graph traversals.

## Design Philosophy

BIM clash detection is fundamentally a relationship-heavy problem:

- **Spatial containment**: An element is contained in a space, which is on a storey, which is in a building, which is on a site. These nested containment relationships form a tree that is queried constantly for filtering and grouping.
- **Clash relationships**: A clash is a relationship between two elements. Clashes cluster around root causes (a misaligned grid affects dozens of element pairs). Finding these clusters is a graph traversal problem.
- **Discipline interactions**: Which discipline pairs generate the most clashes? Which specific model-to-model combinations are problematic? These are relationship analytics.
- **Resolution pathways**: "How was a similar clash resolved in a past project?" requires traversing from the current clash through element types, discipline pairs, spatial contexts, and historical resolutions -- a natural graph query.
- **Impact analysis**: "If we move this beam, which clashes are resolved and which new clashes might appear?" requires traversing spatial neighborhoods.

Relational databases handle these queries through multi-table joins that become expensive at scale. Graph databases traverse these relationships in constant time per hop, making them dramatically faster for connected queries on BIM data.

---

## Neo4j Graph Model

### Node Types and Properties

```cypher
// ============================================================
// SPATIAL HIERARCHY NODES
// ============================================================

// Project: top-level container
(:Project {
    id: "uuid",
    name: "Springfield Hospital",
    code: "SPH-2026",
    status: "active",
    units: "meters",
    projectType: "hospital",
    organizationId: "uuid",
    createdAt: datetime()
})

// Site within a project
(:Site {
    id: "uuid",
    ifcGlobalId: "0YvctVUKr0kugbFTf53O9L",
    name: "Main Campus",
    projectId: "uuid"
})

// Building on a site
(:Building {
    id: "uuid",
    ifcGlobalId: "1a2b3c4d5e6f7g8h9i0j1k",
    name: "Building A - Main Hospital",
    projectId: "uuid"
})

// Storey/level within a building
(:Storey {
    id: "uuid",
    ifcGlobalId: "2b3c4d5e6f7g8h9i0j1k2l",
    name: "Level 2",
    elevation: 8.4,
    projectId: "uuid"
})

// Space/room within a storey
(:Space {
    id: "uuid",
    ifcGlobalId: "3c4d5e6f7g8h9i0j1k2l3m",
    name: "Operating Room 201",
    spaceType: "OperatingRoom",
    area: 45.0,
    projectId: "uuid"
})

// Zone: user-defined grouping that can span storeys
(:Zone {
    id: "uuid",
    name: "Wing A",
    storeys: ["Level 1", "Level 2", "Level 3"],
    projectId: "uuid"
})

// Grid intersection point for spatial reference
(:GridIntersection {
    id: "uuid",
    gridX: "C",
    gridY: "3",
    label: "C-3",
    x: 24.0,
    y: 18.0,
    projectId: "uuid"
})

// ============================================================
// DISCIPLINE AND MODEL NODES
// ============================================================

(:Discipline {
    id: "uuid",
    name: "Mechanical",
    abbreviation: "MEC",
    color: "#FF6B35",
    projectId: "uuid"
})

(:ModelFile {
    id: "uuid",
    name: "MEP_Level2.ifc",
    sourceApplication: "Autodesk Revit 2025",
    sourceFormat: "IFC4",
    projectId: "uuid"
})

(:ModelVersion {
    id: "uuid",
    versionNumber: 3,
    elementCount: 1204,
    processingStatus: "ready",
    ifcProjectGuid: "0YvctVUKr0kugbFTf53O9L",
    checksum: "sha256:abc123...",
    createdAt: datetime()
})

// ============================================================
// BIM ELEMENT NODES
// ============================================================

(:Element {
    id: "uuid",
    ifcGlobalId: "0YvctVUKr0kugbFTf53O9L",
    ifcEntityType: "IfcDuctSegment",
    name: "Rectangular Duct - 600x400",
    objectType: "Rectangular Duct:600x400",

    // Bounding box (stored as individual properties for indexing)
    bboxMinX: 12.0, bboxMinY: 8.0, bboxMinZ: 6.0,
    bboxMaxX: 18.5, bboxMaxY: 8.6, bboxMaxZ: 6.6,

    // Centroid for proximity queries
    centroidX: 15.25, centroidY: 8.3, centroidZ: 6.3,

    // Key quantities
    volume: 0.45,
    length: 6.5,

    // Material
    material: "Galvanized Steel",

    // Classification
    classificationSystem: "Uniclass 2015",
    classificationCode: "Ss_60_40_33",
    classificationName: "Rectangular ductwork systems",

    // System membership
    systemName: "AHU-2 Supply",
    systemType: "HVAC"
})

// Element property set (separate node to avoid property explosion on Element)
(:PropertySet {
    name: "Pset_DuctSegmentTypeCommon",
    // All properties as node properties
    Shape: "RECTANGULAR",
    NominalWidth: 600.0,
    NominalHeight: 400.0,
    InteriorRoughnessCoefficient: 0.0003,
    WorkingPressure: 1500.0
})

// Quantity set
(:QuantitySet {
    name: "Qto_DuctSegmentBaseQuantities",
    Length: 6.5,
    GrossCrossSectionArea: 0.24,
    NetCrossSectionArea: 0.22,
    OuterSurfaceArea: 13.0,
    GrossWeight: 85.0
})

// ============================================================
// CLASH DETECTION NODES
// ============================================================

(:ClashTest {
    id: "uuid",
    name: "MEP vs Structural - Level 2",
    testType: "hard",
    toleranceMm: 0.0,
    status: "completed",
    totalClashes: 156,
    newClashes: 23,
    activeClashes: 89,
    resolvedClashes: 44,
    lastRunAt: datetime(),
    projectId: "uuid"
})

(:ClashTestRun {
    id: "uuid",
    runNumber: 7,
    status: "completed",
    triggeredBy: "model_upload",
    totalClashes: 156,
    clashesAppeared: 23,
    clashesDisappeared: 12,
    elementsCheckedA: 1204,
    elementsCheckedB: 342,
    durationMs: 9500,
    startedAt: datetime(),
    completedAt: datetime()
})

(:Clash {
    hash: "sha256:abc123def456...",
    number: 42,
    distanceMm: -42.3,
    intersectionVolumeMm3: 145230.5,
    penetrationDepthMm: 42.3,

    // Clash point
    pointX: 12.65, pointY: 8.52, pointZ: 6.35,

    // Lifecycle state
    status: "active",
    priority: "high",
    firstSeenAt: datetime(),

    // AI scoring
    aiRelevanceScore: 0.87,
    aiSeverityScore: 0.72,
    aiSuggestedResolution: "Reroute duct below beam B-234 with 50mm offset",
    aiPredictedReworkCostMax: 8000,

    // Grid reference for human context
    gridReference: "C-3, Level 2"
})

(:ClashCluster {
    id: "uuid",
    name: "AHU-2 duct routing conflicts at Level 2",
    clusterType: "ai_root_cause",
    clashCount: 12,
    rootCause: "Misaligned structural grid at column line C",
    aiConfidence: 0.89,
    status: "open"
})

// ============================================================
// ISSUE / BCF NODES
// ============================================================

(:Issue {
    id: "uuid",
    bcfGuid: "uuid",
    serverAssignedId: "ISS-0042",
    title: "Duct penetrates beam at grid C-3, Level 2",
    topicType: "Clash",
    topicStatus: "Open",
    priority: "Major",
    creationAuthor: "j.smith@example.com",
    creationDate: datetime()
})

(:Viewpoint {
    id: "uuid",
    bcfGuid: "uuid",
    cameraType: "perspective",
    cameraViewPointX: 12.5, cameraViewPointY: 8.4, cameraViewPointZ: 9.2,
    cameraDirectionX: 0.3, cameraDirectionY: -0.5, cameraDirectionZ: -0.1,
    fieldOfView: 60.0,
    snapshotPath: "s3://snapshots/viewpoint-uuid.png"
})

(:Comment {
    id: "uuid",
    bcfGuid: "uuid",
    author: "j.smith@example.com",
    text: "Confirmed with structural engineer. Beam cannot be moved. Rerouting duct.",
    createdAt: datetime()
})

// ============================================================
// AI/ML NODES
// ============================================================

(:MLModel {
    id: "uuid",
    modelType: "relevance_classifier",
    version: "v2.3.1",
    framework: "xgboost",
    accuracy: 0.91,
    precision: 0.88,
    recall: 0.93,
    isActive: true,
    artifactPath: "s3://ml-models/relevance/v2.3.1/model.xgb",
    trainedAt: datetime()
})

(:Resolution {
    id: "uuid",
    type: "design_change",
    description: "Rerouted duct below beam, maintaining 50mm clearance",
    actualReworkCost: 4200,
    timeToResolveHours: 72.5,
    rfiReference: "RFI-234"
})
```

### Relationship Types

```cypher
// ============================================================
// SPATIAL CONTAINMENT HIERARCHY
// ============================================================

(:Project)-[:HAS_SITE]->(:Site)
(:Site)-[:HAS_BUILDING]->(:Building)
(:Building)-[:HAS_STOREY]->(:Storey)
(:Storey)-[:HAS_SPACE]->(:Space)
(:Storey)-[:PART_OF_ZONE]->(:Zone)
(:Storey)-[:HAS_GRID_INTERSECTION]->(:GridIntersection)

// ============================================================
// MODEL RELATIONSHIPS
// ============================================================

(:Project)-[:HAS_DISCIPLINE]->(:Discipline)
(:Project)-[:HAS_MODEL]->(:ModelFile)
(:ModelFile)-[:BELONGS_TO_DISCIPLINE]->(:Discipline)
(:ModelFile)-[:HAS_VERSION]->(:ModelVersion)

// ============================================================
// ELEMENT RELATIONSHIPS (the core of the graph)
// ============================================================

// Spatial containment: which storey/space contains this element
(:Element)-[:CONTAINED_IN]->(:Storey)
(:Element)-[:CONTAINED_IN]->(:Space)
(:Element)-[:NEAR_GRID {distance: 2.5}]->(:GridIntersection)

// Model provenance
(:Element)-[:FROM_VERSION]->(:ModelVersion)
(:Element)-[:OF_DISCIPLINE]->(:Discipline)

// Property and quantity sets
(:Element)-[:HAS_PROPERTIES]->(:PropertySet)
(:Element)-[:HAS_QUANTITIES]->(:QuantitySet)

// Spatial proximity (precomputed during model processing)
(:Element)-[:ADJACENT_TO {distance: 0.05}]->(:Element)
(:Element)-[:WITHIN_CLEARANCE {distance: 0.15, requiredClearance: 0.25}]->(:Element)

// Structural/MEP system relationships (parsed from IFC)
(:Element)-[:PART_OF_SYSTEM {systemName: "AHU-2 Supply"}]->(:Element)  // duct-to-duct connections
(:Element)-[:CONNECTED_TO]->(:Element)   // port-based connections
(:Element)-[:HOSTED_BY]->(:Element)      // e.g., window hosted by wall

// ============================================================
// CLASH RELATIONSHIPS
// ============================================================

(:Project)-[:HAS_CLASH_TEST]->(:ClashTest)
(:ClashTest)-[:HAS_RUN]->(:ClashTestRun)
(:ClashTestRun)-[:DETECTED]->(:Clash)

// The core clash relationship: connects two elements through the clash
(:Element)<-[:CLASHES_WITH_A]-(:Clash)-[:CLASHES_WITH_B]->(:Element)

// Clash location
(:Clash)-[:LOCATED_ON]->(:Storey)
(:Clash)-[:NEAR_GRID {distance: 1.2}]->(:GridIntersection)
(:Clash)-[:IN_ZONE]->(:Zone)

// Clash grouping
(:Clash)-[:MEMBER_OF]->(:ClashCluster)

// Clash lifecycle
(:Clash)-[:ASSIGNED_TO]->(:User)
(:Clash)-[:RESOLVED_WITH]->(:Resolution)

// AI scoring provenance
(:Clash)-[:SCORED_BY]->(:MLModel)

// Historical resolution similarity (key graph advantage)
(:Clash)-[:SIMILAR_TO {similarity: 0.92}]->(:Clash)

// ============================================================
// ISSUE / BCF RELATIONSHIPS
// ============================================================

(:Project)-[:HAS_ISSUE]->(:Issue)
(:Issue)-[:ORIGINATED_FROM]->(:Clash)
(:Issue)-[:HAS_VIEWPOINT]->(:Viewpoint)
(:Issue)-[:HAS_COMMENT]->(:Comment)
(:Issue)-[:ASSIGNED_TO]->(:User)
(:Viewpoint)-[:HIGHLIGHTS]->(:Element)
(:Comment)-[:REFERENCES_VIEWPOINT]->(:Viewpoint)
(:Comment)-[:AUTHORED_BY]->(:User)

// ============================================================
// CROSS-PROJECT KNOWLEDGE GRAPH
// ============================================================

// Element type similarity across projects
(:Element)-[:SAME_TYPE_AS]->(:Element)  // same IfcEntityType + ObjectType

// Resolution pattern graph: enables ML-free "similar resolution" queries
(:Resolution)-[:APPLIED_TO_TYPE_PAIR {
    ifcTypeA: "IfcDuctSegment",
    ifcTypeB: "IfcBeam",
    disciplineA: "Mechanical",
    disciplineB: "Structural"
}]->(:Resolution)
```

### Neo4j Indexes and Constraints

```cypher
// ============================================================
// CONSTRAINTS (enforce uniqueness and existence)
// ============================================================

CREATE CONSTRAINT project_id IF NOT EXISTS FOR (p:Project) REQUIRE p.id IS UNIQUE;
CREATE CONSTRAINT element_id IF NOT EXISTS FOR (e:Element) REQUIRE e.id IS UNIQUE;
CREATE CONSTRAINT element_ifc_guid IF NOT EXISTS FOR (e:Element) REQUIRE (e.ifcGlobalId, e.modelVersionId) IS UNIQUE;
CREATE CONSTRAINT clash_hash IF NOT EXISTS FOR (c:Clash) REQUIRE c.hash IS UNIQUE;
CREATE CONSTRAINT clash_test_id IF NOT EXISTS FOR (ct:ClashTest) REQUIRE ct.id IS UNIQUE;
CREATE CONSTRAINT model_version_id IF NOT EXISTS FOR (mv:ModelVersion) REQUIRE mv.id IS UNIQUE;
CREATE CONSTRAINT issue_id IF NOT EXISTS FOR (i:Issue) REQUIRE i.id IS UNIQUE;
CREATE CONSTRAINT issue_bcf_guid IF NOT EXISTS FOR (i:Issue) REQUIRE i.bcfGuid IS UNIQUE;
CREATE CONSTRAINT storey_id IF NOT EXISTS FOR (s:Storey) REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT ml_model_id IF NOT EXISTS FOR (m:MLModel) REQUIRE m.id IS UNIQUE;
CREATE CONSTRAINT resolution_id IF NOT EXISTS FOR (r:Resolution) REQUIRE r.id IS UNIQUE;

// ============================================================
// INDEXES (performance optimization)
// ============================================================

// Element indexes for clash detection queries
CREATE INDEX element_type IF NOT EXISTS FOR (e:Element) ON (e.ifcEntityType);
CREATE INDEX element_storey IF NOT EXISTS FOR (e:Element) ON (e.storeyName);
CREATE INDEX element_discipline IF NOT EXISTS FOR (e:Element) ON (e.disciplineAbbreviation);
CREATE INDEX element_system IF NOT EXISTS FOR (e:Element) ON (e.systemName);

// Spatial indexes using Neo4j point type
CREATE POINT INDEX element_centroid IF NOT EXISTS FOR (e:Element) ON (e.centroid);

// Clash indexes
CREATE INDEX clash_status IF NOT EXISTS FOR (c:Clash) ON (c.status);
CREATE INDEX clash_priority IF NOT EXISTS FOR (c:Clash) ON (c.priority);
CREATE INDEX clash_relevance IF NOT EXISTS FOR (c:Clash) ON (c.aiRelevanceScore);
CREATE INDEX clash_test_project IF NOT EXISTS FOR (ct:ClashTest) ON (ct.projectId);

// Issue indexes
CREATE INDEX issue_status IF NOT EXISTS FOR (i:Issue) ON (i.topicStatus);
CREATE INDEX issue_project IF NOT EXISTS FOR (i:Issue) ON (i.projectId);

// Full-text search indexes
CREATE FULLTEXT INDEX element_search IF NOT EXISTS FOR (e:Element) ON EACH [e.name, e.material, e.systemName];
CREATE FULLTEXT INDEX clash_search IF NOT EXISTS FOR (c:Clash) ON EACH [c.gridReference, c.aiSuggestedResolution];
CREATE FULLTEXT INDEX issue_search IF NOT EXISTS FOR (i:Issue) ON EACH [i.title, i.description];
```

---

## PostgreSQL Sidecar (Transactional Workflow Data)

```sql
-- ============================================================
-- POSTGRESQL SIDECAR
-- Handles ACID transactions for user management, authentication,
-- project configuration, BCF exchange, and audit logging.
-- These entities have simple structures and need strong consistency.
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
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

CREATE TABLE project_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL,  -- references Neo4j Project node
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    discipline      VARCHAR(100),
    UNIQUE(project_id, user_id)
);

-- BCF exchange log
CREATE TABLE bcf_exchanges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    bcf_version     VARCHAR(10) NOT NULL,
    filename        VARCHAR(255) NOT NULL,
    topics_count    INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'completed',
    details         JSONB NOT NULL DEFAULT '{}',
    performed_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Report templates and scheduling
CREATE TABLE report_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID,
    name            VARCHAR(255) NOT NULL,
    report_type     VARCHAR(50) NOT NULL,
    format          VARCHAR(20) NOT NULL DEFAULT 'pdf',
    query_definition JSONB NOT NULL,  -- Cypher query template for the report
    schedule_cron   VARCHAR(100),
    recipients      TEXT[],
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Audit log
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    project_id      UUID,
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       VARCHAR(100) NOT NULL,  -- UUID or neo4j node id
    action          VARCHAR(50) NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_project ON audit_log(project_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

-- Notification queue
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    project_id      UUID,
    notification_type VARCHAR(100) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    data            JSONB NOT NULL DEFAULT '{}',
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read, created_at DESC);

-- File storage references
CREATE TABLE file_references (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    file_type       VARCHAR(50) NOT NULL, -- ifc_model, snapshot, attachment, geometry
    filename        VARCHAR(255) NOT NULL,
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    checksum_sha256 VARCHAR(64),
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_file_refs_entity ON file_references(entity_type, entity_id);
```

---

## Key Graph Queries

### Clash Detection Queries (the graph advantage)

```cypher
// 1. Find all active clashes for a project, with full context
MATCH (p:Project {id: $projectId})-[:HAS_CLASH_TEST]->(ct:ClashTest)-[:HAS_RUN]->(run:ClashTestRun)-[:DETECTED]->(clash:Clash)
WHERE clash.status IN ['new', 'active']
MATCH (clash)-[:CLASHES_WITH_A]->(elemA:Element)-[:OF_DISCIPLINE]->(discA:Discipline)
MATCH (clash)-[:CLASHES_WITH_B]->(elemB:Element)-[:OF_DISCIPLINE]->(discB:Discipline)
MATCH (clash)-[:LOCATED_ON]->(storey:Storey)
OPTIONAL MATCH (clash)-[:ASSIGNED_TO]->(assignee:User)
RETURN clash, elemA, elemB, discA.name AS disciplineA, discB.name AS disciplineB,
       storey.name AS storeyName, assignee.displayName AS assignedTo
ORDER BY clash.aiRelevanceScore DESC
LIMIT 50

// 2. Find clash clusters: groups of clashes sharing common elements
// (This query finds "root cause" patterns - e.g., one misaligned beam
// causes 15 clashes with different ducts)
MATCH (elem:Element)<-[:CLASHES_WITH_A|CLASHES_WITH_B]-(clash:Clash {status: 'active'})
WITH elem, collect(clash) AS clashes, count(clash) AS clashCount
WHERE clashCount >= 3
MATCH (elem)-[:CONTAINED_IN]->(storey:Storey)
RETURN elem.ifcEntityType, elem.name, storey.name,
       clashCount, [c IN clashes | c.hash] AS clashHashes
ORDER BY clashCount DESC

// 3. Impact analysis: "If we move element X, which clashes are affected?"
MATCH (target:Element {ifcGlobalId: $elementGlobalId})
MATCH (target)<-[:CLASHES_WITH_A|CLASHES_WITH_B]-(clash:Clash)
MATCH (clash)-[:CLASHES_WITH_A|CLASHES_WITH_B]->(other:Element)
WHERE other <> target
MATCH (other)-[:OF_DISCIPLINE]->(disc:Discipline)
MATCH (clash)-[:LOCATED_ON]->(storey:Storey)
RETURN clash.hash, clash.status, clash.priority,
       other.ifcEntityType, other.name, disc.name AS discipline,
       storey.name AS storey, clash.distanceMm
ORDER BY clash.priority DESC, clash.aiSeverityScore DESC

// 4. Cross-project resolution recommendation:
// "How were similar clashes resolved in past projects?"
MATCH (currentClash:Clash {hash: $clashHash})
MATCH (currentClash)-[:CLASHES_WITH_A]->(elemA:Element)
MATCH (currentClash)-[:CLASHES_WITH_B]->(elemB:Element)
MATCH (currentClash)-[:LOCATED_ON]->(storey:Storey)

// Find historical clashes with same element type pair
MATCH (historicalClash:Clash)-[:CLASHES_WITH_A]->(histA:Element)
MATCH (historicalClash)-[:CLASHES_WITH_B]->(histB:Element)
WHERE historicalClash.status = 'resolved'
  AND histA.ifcEntityType = elemA.ifcEntityType
  AND histB.ifcEntityType = elemB.ifcEntityType
  AND historicalClash <> currentClash

// Get the resolution
MATCH (historicalClash)-[:RESOLVED_WITH]->(resolution:Resolution)
MATCH (historicalClash)-[:LOCATED_ON]->(histStorey:Storey)

// Calculate similarity (simple: same type pair + similar distance)
WITH currentClash, historicalClash, resolution,
     abs(currentClash.distanceMm - historicalClash.distanceMm) AS distanceDiff,
     histStorey.name AS historicalStorey
WHERE distanceDiff < 100  // within 100mm penetration similarity
RETURN resolution.type, resolution.description,
       resolution.actualReworkCost, resolution.timeToResolveHours,
       historicalStorey, historicalClash.hash,
       1.0 / (1.0 + distanceDiff/100.0) AS similarityScore
ORDER BY similarityScore DESC
LIMIT 5

// 5. Discipline interaction analysis:
// "Which discipline pairs have the most unresolved clashes?"
MATCH (p:Project {id: $projectId})-[:HAS_CLASH_TEST]->(ct:ClashTest)
MATCH (ct)-[:HAS_RUN]->(run:ClashTestRun)-[:DETECTED]->(clash:Clash)
WHERE clash.status IN ['new', 'active']
MATCH (clash)-[:CLASHES_WITH_A]->(elemA:Element)-[:OF_DISCIPLINE]->(discA:Discipline)
MATCH (clash)-[:CLASHES_WITH_B]->(elemB:Element)-[:OF_DISCIPLINE]->(discB:Discipline)
WITH discA.name AS disciplineA, discB.name AS disciplineB,
     count(clash) AS clashCount,
     avg(clash.aiSeverityScore) AS avgSeverity,
     collect(DISTINCT clash.storeyName) AS affectedStoreys
RETURN disciplineA, disciplineB, clashCount, avgSeverity, affectedStoreys
ORDER BY clashCount DESC

// 6. Spatial neighborhood query:
// "Find all elements within 2 meters of this clash point"
MATCH (elem:Element)
WHERE point.distance(
    point({x: elem.centroidX, y: elem.centroidY, z: elem.centroidZ}),
    point({x: $clashPointX, y: $clashPointY, z: $clashPointZ})
) < 2.0
MATCH (elem)-[:OF_DISCIPLINE]->(disc:Discipline)
MATCH (elem)-[:CONTAINED_IN]->(storey:Storey)
RETURN elem.ifcEntityType, elem.name, disc.name, storey.name,
       point.distance(
           point({x: elem.centroidX, y: elem.centroidY, z: elem.centroidZ}),
           point({x: $clashPointX, y: $clashPointY, z: $clashPointZ})
       ) AS distanceFromClash
ORDER BY distanceFromClash

// 7. Recurring pattern detection:
// "Find storeys where the same type of clash keeps reappearing across runs"
MATCH (ct:ClashTest {id: $clashTestId})-[:HAS_RUN]->(run:ClashTestRun)-[:DETECTED]->(clash:Clash)
MATCH (clash)-[:CLASHES_WITH_A]->(elemA:Element)
MATCH (clash)-[:CLASHES_WITH_B]->(elemB:Element)
MATCH (clash)-[:LOCATED_ON]->(storey:Storey)
WITH storey.name AS storeyName,
     elemA.ifcEntityType + '-' + elemB.ifcEntityType AS typePair,
     count(DISTINCT run) AS runsWithClash,
     count(clash) AS totalClashInstances
WHERE runsWithClash >= 3  // appeared in 3+ runs = recurring
RETURN storeyName, typePair, runsWithClash, totalClashInstances
ORDER BY runsWithClash DESC, totalClashInstances DESC

// 8. System-level clash analysis:
// "How many clashes involve the AHU-2 supply duct system?"
MATCH (elem:Element {systemName: "AHU-2 Supply"})
MATCH (elem)<-[:CLASHES_WITH_A|CLASHES_WITH_B]-(clash:Clash)
WHERE clash.status IN ['new', 'active']
MATCH (clash)-[:CLASHES_WITH_A|CLASHES_WITH_B]->(other:Element)
WHERE other <> elem
MATCH (other)-[:OF_DISCIPLINE]->(otherDisc:Discipline)
WITH otherDisc.name AS conflictingDiscipline,
     count(clash) AS clashCount,
     avg(clash.aiSeverityScore) AS avgSeverity,
     collect(DISTINCT other.ifcEntityType) AS conflictingTypes
RETURN conflictingDiscipline, clashCount, avgSeverity, conflictingTypes
ORDER BY clashCount DESC
```

### Graph-Powered AI Features

```cypher
// Build feature vectors for ML training from graph structure
// (Graph features that are impossible to compute efficiently in SQL)

// Feature: number of clashes within 5 meters of this clash
MATCH (thisClash:Clash {hash: $clashHash})
MATCH (nearby:Clash)
WHERE nearby <> thisClash
  AND nearby.status IN ['new', 'active']
  AND point.distance(
      point({x: thisClash.pointX, y: thisClash.pointY, z: thisClash.pointZ}),
      point({x: nearby.pointX, y: nearby.pointY, z: nearby.pointZ})
  ) < 5.0
RETURN count(nearby) AS nearbyClashCount

// Feature: shortest path between clashing elements through spatial structure
MATCH (elemA:Element {ifcGlobalId: $globalIdA})
MATCH (elemB:Element {ifcGlobalId: $globalIdB})
MATCH path = shortestPath((elemA)-[:CONTAINED_IN|ADJACENT_TO|CONNECTED_TO*..5]-(elemB))
RETURN length(path) AS spatialPathLength, [n IN nodes(path) | labels(n)[0]] AS pathNodeTypes

// Feature: historical resolution rate for this element type pair in this spatial context
MATCH (storey:Storey {name: $storeyName})
MATCH (storey)<-[:LOCATED_ON]-(historicalClash:Clash {status: 'resolved'})
MATCH (historicalClash)-[:CLASHES_WITH_A]->(hA:Element {ifcEntityType: $typeA})
MATCH (historicalClash)-[:CLASHES_WITH_B]->(hB:Element {ifcEntityType: $typeB})
MATCH (historicalClash)-[:RESOLVED_WITH]->(res:Resolution)
RETURN count(historicalClash) AS historicalCount,
       avg(res.timeToResolveHours) AS avgResolutionTime,
       collect(res.type) AS resolutionTypes
```

---

## Data Synchronization Between Neo4j and PostgreSQL

```python
# Sync service that keeps Neo4j and PostgreSQL in sync
# Uses outbox pattern: PostgreSQL writes trigger Neo4j updates

class GraphSyncService:
    """
    Bidirectional sync between PostgreSQL (transactional) and Neo4j (graph).

    PostgreSQL -> Neo4j: User creates/updates project, assigns clash, changes status
    Neo4j -> PostgreSQL: Clash detection engine writes results to Neo4j, summary
                         stats synced back for dashboard queries
    """

    async def sync_clash_status_change(self, clash_hash: str, new_status: str,
                                        changed_by_user_id: str, reason: str):
        """Update clash status in Neo4j when changed via the REST API (PostgreSQL-first)."""
        async with self.neo4j.session() as session:
            await session.run("""
                MATCH (c:Clash {hash: $hash})
                SET c.status = $status,
                    c.updatedAt = datetime()
                WITH c
                OPTIONAL MATCH (c)-[old:ASSIGNED_TO]->()
                DELETE old
                WITH c
                MATCH (u:User {id: $userId})
                MERGE (c)-[:LAST_MODIFIED_BY]->(u)
            """, hash=clash_hash, status=new_status, userId=changed_by_user_id)

        # Also write to PostgreSQL audit log
        await self.pg.execute("""
            INSERT INTO audit_log (project_id, user_id, entity_type, entity_id, action, changes)
            VALUES ($1, $2, 'clash', $3, 'status_change', $4)
        """, project_id, changed_by_user_id, clash_hash,
             json.dumps({"status": {"to": new_status}, "reason": reason}))

    async def sync_clash_detection_results(self, clash_test_run_id: str):
        """After clash detection completes in Neo4j, sync summary to PostgreSQL."""
        async with self.neo4j.session() as session:
            result = await session.run("""
                MATCH (run:ClashTestRun {id: $runId})-[:DETECTED]->(c:Clash)
                RETURN count(c) AS total,
                       count(CASE WHEN c.status = 'new' THEN 1 END) AS newCount,
                       count(CASE WHEN c.status = 'active' THEN 1 END) AS activeCount,
                       count(CASE WHEN c.status = 'resolved' THEN 1 END) AS resolvedCount
            """, runId=clash_test_run_id)
            stats = result.single()

        # Sync summary to PostgreSQL for dashboard queries
        # (PostgreSQL is faster for simple aggregation queries used by dashboards)
```

---

## Pros and Cons

### Pros

1. **Natural relationship modeling**: BIM data is inherently a graph -- elements connected to spaces, spaces to storeys, clashes connecting element pairs. The graph model matches the domain mental model directly, making queries intuitive and the schema self-documenting.

2. **Cluster and pattern detection**: Finding groups of clashes that share a root cause (e.g., all clashes caused by a single misaligned beam) is a natural graph traversal. In SQL, this requires recursive CTEs or self-joins that are both harder to write and slower to execute.

3. **Cross-project knowledge graph**: The ability to traverse from a current clash through element type pairs and spatial contexts to find similar resolved clashes in other projects is a unique graph capability. This "resolution recommendation" feature becomes more powerful as more projects add data to the graph.

4. **Impact analysis**: "What happens if we move this element?" requires finding all clashes involving it, all connected elements, and all downstream effects. Graph traversal handles this elegantly; relational joins would require complex recursive queries.

5. **Variable-depth traversal**: Queries like "find all elements connected to this element within 3 hops through any relationship type" have no fixed join depth. Graph databases handle variable-depth traversals natively with `*..N` syntax.

6. **Schema flexibility**: Adding new node properties or relationship types requires no schema migration. New IFC entity types, new property sets, new AI scoring dimensions can be added as properties without ALTER TABLE operations.

7. **Spatial proximity as relationships**: Pre-computing ADJACENT_TO and WITHIN_CLEARANCE relationships during model processing means that spatial proximity queries at coordination time are simple relationship traversals, not geometric computations.

### Cons

1. **Dual-database complexity**: Maintaining consistency between Neo4j (graph) and PostgreSQL (transactional) requires a synchronization layer. This introduces failure modes, eventual consistency windows, and operational overhead that a single-database solution avoids.

2. **Lack of ACID transactions across stores**: A status change must update both Neo4j and PostgreSQL. If one succeeds and the other fails, the system is in an inconsistent state. Implementing distributed transactions (2PC) or compensating actions adds significant complexity.

3. **Limited aggregate query performance**: Neo4j is optimized for traversals, not for "COUNT all clashes grouped by status and priority." Dashboard-style aggregation queries that scan many nodes are slower in Neo4j than in PostgreSQL, which is why the PostgreSQL sidecar is needed.

4. **Steep learning curve**: Cypher query language and graph-thinking are unfamiliar to most developers. The AEC software development community overwhelmingly uses SQL. Recruiting contributors for an open-source project is harder when the primary data store requires specialized knowledge.

5. **Bulk loading performance**: Importing 500,000 BIM elements with their relationships is a bulk operation that Neo4j handles less efficiently than PostgreSQL's COPY command. The initial model processing pipeline may be bottlenecked by Neo4j write performance.

6. **Operational maturity**: Neo4j's backup, monitoring, and high-availability tooling is less mature than PostgreSQL's. Fewer hosting options, fewer managed service providers, and a smaller pool of operational expertise in the market.

7. **Cost at scale**: Neo4j Enterprise Edition (required for clustering, online backup, and role-based access control) is commercially licensed. The Community Edition has limitations on database size and lacks features needed for production deployment. This conflicts with the project's open-source positioning.

8. **No native 3D spatial indexing**: While Neo4j has point indexes for 2D geographic data, it lacks true 3D spatial indexing. Bounding box intersection queries for clash detection still require application-level computation or fall back to PostgreSQL/PostGIS.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Graph Database | Neo4j Community 5.x (or Memgraph for open-source alternative) | Core relationship store; Memgraph is MIT-licensed |
| Transactional Store | PostgreSQL 16+ | User management, workflow, audit, BCF exchange |
| Spatial Queries | PostGIS on PostgreSQL | 3D bounding box intersection for clash detection preprocessing |
| Sync Layer | Debezium CDC or custom outbox pattern | Keep Neo4j and PostgreSQL in sync |
| Object Storage | S3/MinIO | IFC files, geometry meshes, snapshots |
| Caching | Redis | Dashboard aggregation cache, session management |
| API Layer | Node.js/TypeScript with neo4j-driver + pg | Dual-database access layer |
| Visualization | Neo4j Browser / Bloom (development), Custom 3D viewer (production) | Graph exploration for debugging and analysis |
| ML Pipeline | Python with neo4j Python driver + scikit-learn/xgboost | Extract graph features for training |

### Alternative: Memgraph Instead of Neo4j

For a fully open-source stack, consider Memgraph (BSL/MIT licensed) as the graph database:
- Compatible with Cypher query language
- In-memory architecture offers faster traversal for real-time queries
- Supports MAGE (Memgraph Advanced Graph Extensions) for built-in graph algorithms
- Lower operational overhead than Neo4j cluster deployment
- Trade-off: smaller community, fewer enterprise features, data must fit in memory

---

## Migration and Scaling Considerations

### Initial Deployment

- Single Neo4j Community instance (16GB RAM, SSD)
- Single PostgreSQL instance (8GB RAM)
- Application-level sync between stores
- Graph data fully in-memory for projects under 1M elements

### Growth Phase (50-200 projects)

- Neo4j with increased heap (32-64GB) for larger graphs
- Read replicas for Neo4j (requires Enterprise or Memgraph)
- PostgreSQL read replicas for dashboard queries
- Introduce Debezium CDC for reliable sync
- Archive completed project subgraphs to separate databases

```cypher
// Subgraph export for project archival
CALL apoc.export.cypher.query(
    "MATCH (n) WHERE n.projectId = $projectId
     OPTIONAL MATCH (n)-[r]->(m) WHERE m.projectId = $projectId
     RETURN n, r, m",
    "/export/project-SPH-2026.cypher",
    {params: {projectId: $projectId}}
)
```

### Enterprise Scale

- Neo4j Fabric for federated graph queries across multiple databases
- Each large project gets its own Neo4j database within the DBMS
- Cross-project knowledge graph maintained as a separate lightweight database
- PostgreSQL with Citus for horizontal scaling of transactional data
- Consider Apache AGE (graph extension for PostgreSQL) to eliminate the dual-database complexity if graph query needs are moderate

### Migration from Relational Systems

1. **Export BIM elements** from PostgreSQL to Neo4j using `neo4j-admin import` (CSV bulk loader)
2. **Build spatial relationships** (CONTAINED_IN, ADJACENT_TO) via post-import Cypher batch
3. **Import clash results** with element references resolved to Neo4j node IDs
4. **Build resolution graph** from historical data to enable cross-project recommendations
5. **Parallel operation** period where both relational and graph stores serve queries
6. **Gradual cutover** of query paths from SQL to Cypher

### Sizing Estimates

| Project Scale | Elements | Relationships | Neo4j Memory | PostgreSQL |
|---|---|---|---|---|
| Small (1 building) | 50K | 500K | 4GB | 2GB |
| Medium (hospital campus) | 500K | 5M | 16GB | 10GB |
| Large (airport) | 2M | 20M | 64GB | 40GB |
| Enterprise (portfolio of 100 projects) | 50M | 500M | 256GB+ cluster | 200GB+ |

The graph grows superlinearly with element count because relationships (ADJACENT_TO, WITHIN_CLEARANCE) grow quadratically in dense spatial regions. Memory planning should account for this with a 10:1 relationship-to-element ratio for BIM data.
