# Standards & API Reference

> Project: BIM Clash Detection & Management · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 16739-1:2024 — Industry Foundation Classes (IFC)**
- URL: https://www.iso.org/standard/84123.html
- The foundational open data schema for sharing building and civil infrastructure information across software applications. IFC is the primary open format for model exchange in BIM workflows. Clash detection tools must ingest and process IFC files to operate in a vendor-neutral pipeline. The latest revision (IFC4X3_ADD2) extends coverage to infrastructure (roads, bridges, rail, ports).

**ISO 19650 — Information Management using Building Information Modelling**
- URL: https://www.iso.org/standard/68078.html (Part 1); https://www.iso.org/standard/68080.html (Part 2)
- The international standard governing how BIM information is created, exchanged, and managed across the project lifecycle. ISO 19650 defines Common Data Environments (CDEs), naming conventions, and information delivery milestones — all of which determine how clash detection results are formally recorded and handed over. Compliance is mandatory on public-sector projects in the UK and many EU countries.

**ISO 29481 — Building Information Models: Information Delivery Manual (IDM)**
- URL: https://www.iso.org/standard/60553.html
- Specifies the process maps and data exchange requirements (Exchange Requirements) that define what model information must be present at each project stage. IDM informs which model elements are in scope for clash detection at each delivery milestone and what properties must be present for a clash to be actionable.

**ISO 12006-3 — Building construction: Organization of information about construction works — Framework for object-oriented information**
- URL: https://www.iso.org/standard/57803.html
- Underpins classification systems (such as Uniclass and OmniClass) that are used to classify BIM objects during clash detection — determining which elements belong to which discipline in a clash test matrix.

---

### buildingSMART International Standards

**BIM Collaboration Format (BCF)**
- URL: https://technical.buildingsmart.org/standards/bcf/
- BCF is the open standard for model-based issue communication. It enables issues (including clash results) to be exchanged between different BIM authoring tools and coordination platforms without proprietary format dependency. BCF consists of XML-structured issue data including viewpoints (camera position), component references (IFC GUIDs), snapshots (PNG), and metadata. The BCF REST API (version 3.0) replaces file-based exchange with a RESTful web service for live issue synchronisation between platforms.

**BCF REST API v3.0**
- URL: https://github.com/buildingSMART/BCF-API (release_3_0 branch)
- The open REST API specification for BCF. Data is exchanged via URL-encoded query parameters and JSON bodies over HTTP. The specification covers endpoints for projects, topics (issues), comments, viewpoints, documents, and extensions. Any conformant platform can push and pull issues without vendor lock-in. BCF Manager plugins for Revit, ArchiCAD, and other tools implement this API client-side.

**Information Delivery Specification (IDS)**
- URL: https://technical.buildingsmart.org/projects/information-delivery-specification-ids/
- A machine-interpretable format for defining what IFC data must be present in a model at a given stage. IDS can be used to validate models before clash detection — ensuring elements have the property sets and classifications needed for meaningful clash tests.

**IFC Schema Specification (IFC 4.3.2)**
- URL: https://ifc43-docs.standards.buildingsmart.org/
- The current normative specification for the IFC data model. Relevant sections for clash detection include IfcElement geometry, IfcRelSpaceBoundary, IfcZone, and property set definitions for MEP and structural elements. All clash detection tools consuming IFC must conform to this schema.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Governs HTTP request/response semantics used by all REST APIs in the BIM coordination space (BCF API, APS API, Revizto API, Trimble Connect API). All cloud-based coordination platforms use HTTP/1.1 or HTTP/2 for API communication.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is used for authentication tokens in the BCF API, Autodesk Platform Services, and other BIM coordination APIs. Platform-level clash data access is controlled via JWT bearer tokens.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Hypermedia linking conventions used in pagination of large clash result sets returned by BCF-API and APS API endpoints.

---

### Data Model & API Specifications

**OpenAPI Specification 3.x**
- URL: https://spec.openapis.org/oas/latest.html
- The APS (Autodesk Platform Services) Model Coordination API, Revizto API, and BCF REST API are all documented using OpenAPI 3.x, enabling code generation for client SDKs in JavaScript, Python, Java, and other languages.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Used for validating BCF API request/response bodies and issue payloads. BCF issue topics, viewpoints, and component references are described as JSON Schema objects.

**CDE-API (buildingSMART)**
- URL: https://technical.buildingsmart.org/projects/opencdapi/
- The open Common Data Environment API specification from buildingSMART. Defines standardised interfaces for document and model management across CDEs such as BIMcollab Nexus, Procore, and ACC. Clash reports and model versions are stored and accessed via CDE-API compliant endpoints.

**COBie (Construction Operations Building Information Exchange)**
- URL: https://www.nibs.org/resources/cobie
- A spreadsheet-compatible data format for asset handover. COBie data is often generated from coordinated BIM models after clash resolution as part of the facility management handover — a downstream deliverable of a completed clash management workflow.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The authentication framework used by Autodesk Platform Services (3-legged and 2-legged OAuth flows), Revizto API, and BIMcollab BCF API. All user-level clash data access via API requires OAuth 2.0 authorisation.

**OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- Used by Autodesk and other SaaS platforms for federated identity and single sign-on (SSO) into BIM coordination platforms.

**TLS 1.3 (RFC 8446)**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- All production REST APIs (BCF, APS, Revizto, BIMcollab) require TLS-encrypted transport. TLS 1.3 is the current recommended minimum for new API clients.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- Relevant for any clash management platform exposing a REST API. Key risks include broken object-level authorisation (accessing another project's clash data), excessive data exposure in clash result payloads, and insecure direct object references for issue IDs.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is potentially relevant for an AI-native clash detection tool. An MCP server could expose:
- Clash result sets as tool-readable resources for AI reasoning over spatial conflicts
- Issue assignment and status update as callable tools for AI-driven coordination agents
- BCF API interaction as an MCP tool for pushing issues to connected platforms

**MCP Specification**
- URL: https://modelcontextprotocol.io/specification
- The MCP specification defines how AI models can interact with external data sources and tools in a standardised way. An MCP server wrapping the BCF API would enable AI agents to query, triage, and resolve BIM coordination issues autonomously.

---

## Similar Products — Developer Documentation & APIs

### Autodesk Platform Services (APS) — Model Coordination API

- **Description:** The cloud-based model coordination and clash detection API from Autodesk, compatible with both ACC and BIM360 projects. Enables programmatic access to clash test configuration, clash results, clash grouping, and issue creation.
- **API Documentation:** https://aps.autodesk.com/en/docs/acc/v1/overview/field-guide/model-coordination/mcfg-clash
- **SDKs/Libraries:** APS SDKs available for JavaScript/Node.js, Python, .NET, and Java at https://aps.autodesk.com/developer/start
- **Developer Guide:** https://aps.autodesk.com/developer/overview/model-coordination
- **Standards:** REST/JSON, OpenAPI 3.x
- **Authentication:** OAuth 2.0 (2-legged for service-to-service, 3-legged for user-context)
- **Sample Code:** https://github.com/autodesk-platform-services/aps-clash-data-view

### Autodesk Navisworks API

- **Description:** .NET and COM API providing programmatic access to all Navisworks Manage functionality including Clash Detective — test configuration, execution, results access, and grouping.
- **API Documentation:** https://aps.autodesk.com/developer/overview/navisworks
- **SDKs/Libraries:** .NET API installed with Navisworks Manage SDK (under `\api\` folder); also accessible via APS Forge for cloud-linked workflows
- **Developer Guide:** Navisworks SDK Developer Guide (PDF, installed with SDK); AEC DevBlog at https://adndevblog.typepad.com/aec/navisworks/
- **Standards:** .NET API; COM Interop for legacy functionality
- **Authentication:** Local desktop application; APS OAuth for cloud-linked features

### BIMcollab BCF API

- **Description:** BIMcollab Nexus implements the buildingSMART BCF REST API v3.0 as its primary integration interface. Issues, viewpoints, projects, and topics can be read and written from any BCF-conformant client.
- **API Documentation:** https://www.bimcollab.com/en/developers/developer-sdk/
- **SDKs/Libraries:** BIMcollab SDK and BCF Manager open-source plugins (GitHub: buildingSMART/BCF-API)
- **Developer Guide:** https://helpcenter.bimcollab.com/en/articles/327345-bimcollab-developer-sdk
- **Standards:** BCF REST API v3.0 (buildingSMART open standard), REST/JSON
- **Authentication:** OAuth 2.0; API key for service integrations

### Revizto API

- **Description:** REST API enabling automation of team management, basic issue operations, clash data access, and audit log export from Revizto projects.
- **API Documentation:** https://help.revizto.com/hc/en-us/articles/9420673106063-Revizto-API-documentation
- **SDKs/Libraries:** No published official SDK; REST/JSON endpoints for direct HTTP integration
- **Developer Guide:** https://developer.revizto.com/ (requires Revizto account); Revizto Academy "API Essentials" course
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0; API access requires Revizto+ subscription tier

### Solibri Developer Platform API

- **Description:** Java-based API for developing custom rules, rule templates, and automation within Solibri Model Checker. Enables programmatic definition of clash checks and compliance rules.
- **API Documentation:** https://solibri.github.io/Developer-Platform/
- **SDKs/Libraries:** Java SDK with Maven build; examples at https://github.com/Solibri/api-examples
- **Developer Guide:** https://solibri.github.io/Developer-Platform/latest/index.html
- **Standards:** Java API; BCF for issue export; IFC for model input
- **Authentication:** Desktop application API (no HTTP auth); cloud integrations via Documents API use OAuth

### Trimble Connect API

- **Description:** REST API for programmatic access to model management, clash detection trigger/results, issue tracking, and task management within Trimble Connect projects.
- **API Documentation:** https://app.connect.trimble.com/tc/developer/
- **SDKs/Libraries:** REST/JSON; no official published SDK; community .NET samples available
- **Developer Guide:** Trimble Developer Network (developer.trimble.com)
- **Standards:** REST/JSON; IFC for model exchange; BCF for issue interoperability
- **Authentication:** OAuth 2.0 (Trimble Identity)

### Newforma Konekt (BCF-API)

- **Description:** Newforma Konekt exposes a BCF REST API endpoint for pushing and pulling coordination issues from connected BIM authoring tools and clash detection platforms.
- **API Documentation:** https://www.newforma.com/app-market/autodesk-navisworks/autodesk-navisworks-newforma-konekt/
- **SDKs/Libraries:** Navisworks add-in plugin; Revit add-in; AVEVA E3D Design plugin
- **Developer Guide:** Newforma developer documentation (login required)
- **Standards:** BCF REST API (buildingSMART); REST/JSON
- **Authentication:** OAuth 2.0 / API key

### IfcOpenShell (Open Source IFC Toolkit)

- **Description:** Open-source C++ and Python library for parsing, writing, and processing IFC files. Includes `ifcopenshell.geom` for geometry processing essential to clash detection, and `bcf` module for reading/writing BCF issue files.
- **API Documentation:** https://docs.ifcopenshell.org/
- **SDKs/Libraries:** Python (pip install ifcopenshell), C++; BCF submodule at https://docs.ifcopenshell.org/bcf.html
- **Developer Guide:** https://academy.ifcopenshell.org/
- **Standards:** IFC (ISO 16739-1:2024); BCF
- **Authentication:** None (local library); LGPL licence

### xBIM Toolkit (Open Source .NET BIM)

- **Description:** Open-source .NET library for creating, reading, and modifying IFC models. Provides geometric processing capabilities usable for building a custom clash detection engine on a .NET/C# stack.
- **API Documentation:** https://docs.xbim.net/
- **SDKs/Libraries:** NuGet packages (Xbim.Essentials, Xbim.Geometry); .NET 6+
- **Developer Guide:** https://docs.xbim.net/quick-start/
- **Standards:** IFC (ISO 16739-1); BCF
- **Authentication:** None (local library); MIT licence (Essentials), CDDL (Geometry)

---

## Notes

- **BCF API v3.0 is the key integration standard** for any open-source or AI-native clash management tool. Conformance with BCF API ensures immediate interoperability with Navisworks, Revit, ArchiCAD, Tekla, Solibri, BIMcollab, and Revizto without requiring proprietary integrations.
- **IFC as the model exchange foundation** avoids dependency on Autodesk NWC/NWD proprietary formats. IfcOpenShell and xBIM are mature open-source libraries that can underpin an open-source clash detection engine.
- **APS Model Coordination API** is the most feature-complete cloud API for programmatic clash management but requires an Autodesk ACC subscription — not suitable as a dependency for a vendor-neutral open-source tool.
- **MCP is an emerging integration point** that could allow AI agents to interact with BCF-based issue trackers and clash results. No production MCP server for BIM coordination exists publicly as of May 2026 — this is an open opportunity.
- **COBie and IDS** are downstream standards that a complete BIM coordination platform should support for asset handover and model validation respectively.
