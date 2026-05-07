# BIM Clash Detection & Management — Feature & Functionality Survey

> Candidate #438 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Autodesk Navisworks Manage | Desktop | Commercial (perpetual/subscription) | https://www.autodesk.com/products/navisworks/ |
| Autodesk Construction Cloud (ACC) Model Coordination | Cloud SaaS | Subscription (BIM Collaborate Pro) | https://construction.autodesk.com/ |
| BIMcollab Nexus + Zoom | Cloud SaaS + Desktop | Freemium → paid tiers | https://www.bimcollab.com/ |
| Revizto | Cloud SaaS | Subscription (Revizto / Revizto+) | https://revizto.com/ |
| Solibri | Desktop + Cloud | Commercial subscription | https://www.solibri.com/ |
| Trimble Connect | Cloud SaaS | Freemium → paid tiers | https://connect.trimble.com/ |
| Newforma Konekt (fmr. BIM Track) | Cloud SaaS | Commercial subscription | https://www.newforma.com/newforma-konekt/ |
| BIMWorkplace | Cloud SaaS | Commercial subscription | https://bimworkplace.com/ |
| AutoBIMRoute | SaaS / Plugin | Commercial | https://autobimroute.com/ |
| Genusys.ai | SaaS / Plugin | Commercial | https://genusys.ai/ |

---

## Feature Analysis by Solution

### Autodesk Navisworks Manage

**Core features**
- Hard clash, soft clash (clearance), and 4D (time-based) clash detection across any combination of discipline models
- Clash Detective: configurable test sets, tolerance settings, and rule-based filtering
- Model federation from Revit, ArchiCAD, AutoCAD, Tekla, and 60+ native formats via NWC/NWD
- Georeferenced clash results with visual snapshot and object-level identification
- Clash grouping by geometry, discipline, grid zone, or custom properties
- Export of clash reports to HTML, XML, and PDF
- Clash assignment to reviewers with status tracking (New, Active, Reviewed, Approved, Resolved)
- .NET API for programmatic access to all Clash Detective data and functions
- TimeLiner 4D simulation for construction sequencing conflict detection
- Quantification and quantity take-off linked to coordinated models

**Differentiating features**
- Industry-standard model aggregation format (NWD/NWC) accepted by virtually every other AEC tool
- Broadest native format support of any clash detection tool
- COM Automation and .NET API allow full scriptable access to clash data and test execution

**UX patterns**
- Tabbed Clash Detective panel within desktop application
- Selection tree for defining clash test pairings
- Matrix-style clash results table with colour-coded status
- Integrated 3D viewport with highlighted clash geometry

**Integration points**
- Autodesk Platform Services (APS) Model Coordination API for cloud-linked workflows
- Newforma Konekt add-in converts Navisworks clash results to tracked issues
- BIMcollab Zoom and Nexus synchronise Navisworks clash results via BCF
- Revizto importer for exporting clash snapshots and linking to issue tracker
- Dynamo scripting for parametric clash workflow automation

**Known gaps**
- No native cloud collaboration — users need ACC or a third-party platform
- Large model performance degrades significantly on complex federated models
- Clash grouping by multiple properties simultaneously requires paid third-party plugins or ACC
- No AI-assisted prioritisation or false-positive filtering built in
- Manual setup required for each new project; no reusable test configuration templates (addressed in ACC but not Navisworks standalone)
- Difficult for non-BIM users to access or interpret results

**Licence / IP notes**
- Proprietary Autodesk software; NWD format is proprietary (NWC is a Navisworks cache format)
- .NET API publicly documented and freely usable for plugin development under Autodesk developer terms

---

### Autodesk Construction Cloud (ACC) Model Coordination

**Core features**
- Automatic clash detection triggered on every model upload (RVT, DWG, NWC, IFC)
- Multi-Property Clash Grouping: group clashes by system type, level, and discipline simultaneously
- Reusable clash check configurations (tolerance, content filter, grouping rules)
- One-click conversion of clash to tracked ACC Issue with context, photos, and location pin
- 2D/3D model viewing in browser — no installed software required
- Clash classification ("issue" vs "not an issue") with comment history
- Integration with ACC Build for field issue tracking continuity
- Role-based access controls for viewing and editing clash data

**Differentiating features**
- Fully browser-based — zero local install for reviewers and contractors
- Automatic re-run of clash detection on every model version upload
- Persistent clash history across model versions for trend analysis

**UX patterns**
- Project-level Model Coordination space within ACC dashboard
- Clash matrix grid view for discipline pair summary
- Clash list with filtering, sorting, and bulk actions
- Native Viewer (Forge/APS Viewer) embedded for 3D context

**Integration points**
- Autodesk Platform Services (APS) Model Coordination API (REST/JSON)
- Autodesk Revit: BIM Collaborate Pro add-in for direct model publishing
- ACC Build for field issues and RFIs
- PowerBI sample connectors via APS API for analytics dashboards

**Known gaps**
- Limited to Autodesk ecosystem for advanced authoring integration
- Less granular clash filtering compared with Navisworks standalone
- Requires BIM Collaborate Pro licence tier (expensive for smaller firms)
- No native 4D clash detection (time-based); this still requires Navisworks

**Licence / IP notes**
- Commercial SaaS; APS API is publicly documented under Autodesk developer terms
- Model data stored in Autodesk cloud; data residency options limited

---

### BIMcollab Nexus + Zoom

**Core features**
- Smart Views: rules-based clash detection directly within BIMcollab Zoom viewer on IFC models
- Smart Issues: clashes auto-converted to issues, status auto-updated on each new model sync
- Issue lifecycle management: assignment, due dates, priority, comments, screenshots
- BCF-standard issue exchange with Revit, ArchiCAD, Tekla, Navisworks, Solibri, and others
- Freemium BCF Manager plugins for Revit, ArchiCAD, Rhino, and Tekla
- Real-time multi-user collaboration on issue status updates
- Issue filtering and bulk editing across large clash sets
- Dashboard with open/closed issue metrics and team workload overview

**Differentiating features**
- Open BCF API means any BCF-compatible tool can push/pull issues without vendor lock-in
- Smart Issues automatically recheck clash status without rerunning tests manually
- Freemium tier with paid BCF Managers makes it accessible to all stakeholders including subcontractors

**UX patterns**
- Web-based issue dashboard and model viewer
- Colour-coded issue status overlay on 3D model
- BCF Manager panel embedded in authoring tool (Revit/ArchiCAD)
- Table view with column filtering for large issue sets

**Integration points**
- BCF-API (buildingSMART open standard REST API)
- BIMcollab Zoom for in-app clash detection on IFC
- Revit, ArchiCAD, Tekla, Navisworks BCF Manager plugins
- AVEVA E3D Design via BCF API
- Trimble Connect and Solibri via BCF

**Known gaps**
- Clash detection engine limited to IFC geometry in Zoom; not as performant as Navisworks for large models
- Advanced analytics and reporting require higher-tier plans
- No built-in 4D clash or time-sequencing analysis
- AI-assisted prioritisation absent

**Licence / IP notes**
- Freemium SaaS; BCF Manager plugins are open source (MIT licence)
- BCF standard is open (buildingSMART); no proprietary lock-in for issue data

---

### Revizto

**Core features**
- Real-time cloud BIM coordination with embedded clash detection and issue tracking
- Collaborative Clash Automation: configurable clash matrix with automatic grouping by zone and level
- Stamp templates for automated issue assignment to responsible disciplines
- 2D/3D integrated viewer: issues tracked in plan, section, and 3D simultaneously
- Mobile app for field issue review and photo capture
- Real-time status updates — all team members see changes immediately
- Issue tracker with custom fields, workflows, and approval stages
- Integrations with Procore, Autodesk, Bentley, Nemetschek, and Trimble authoring tools (30+ plugins)
- Power BI integration for clash analytics and reporting (Revizto+ tier)

**Differentiating features**
- Simultaneous 2D and 3D issue location context — unique among clash platforms
- Real-time collaboration without re-upload cycles
- Proven 50% reduction in coordination time reported by Arcadis, Jacobs, AECOM
- OpenBIM-agnostic: plugins for all major authoring ecosystems

**UX patterns**
- Persistent project workspace where all disciplines always see the latest model and issues
- Clash matrix configuration panel with named test pairs
- Issue card with linked 3D viewpoint, 2D location, assignee, and history
- Mobile-first design for field access

**Integration points**
- Revizto REST API (developer.revizto.com) for issue management, team management, clash data, and audit logs
- Plugin connectors for Revit, Navisworks, ArchiCAD, Tekla, AutoCAD, Bentley tools
- Procore integration for project management
- ACC/BIM360 integration via Autodesk App Store

**Known gaps**
- API access and Power BI integration locked to premium Revizto+ tier
- Clash detection engine is less configurable than Navisworks Clash Detective
- Pricing not publicly listed — sales-led
- No open BCF API exposure natively (relies on proprietary Revizto API)

**Licence / IP notes**
- Proprietary SaaS; no open-source components identified
- Revizto API documentation publicly available but requires account

---

### Solibri

**Core features**
- Rules-based model checking beyond geometry: code compliance, data completeness, classification validation
- Highly configurable rule sets (geometry, IFC properties, spatial relationships)
- Solibri Office: deep QA checking for architects, structural, and MEP designers
- Solibri CheckPoint (2025): cloud-based quality checking for native Revit and IFC
- Issue communication using BCF standard
- Communication module for distributing issues to team members
- Model comparison to detect changes between versions
- Quantity extraction linked to model elements
- Ruleset marketplace via Solibri Lab for community-contributed and certified rules

**Differentiating features**
- Only platform combining geometry clash detection with full BIM data quality/compliance checking
- Rule customisation depth unmatched — clients and authorities can supply formal checking rules
- 3.2 billion issues detected in 2024 across its user base (scale indicator)
- Solibri Lab program for custom API rule development

**UX patterns**
- Desktop-first with structured left-panel rule organisation
- Results presented in categorised issue groups with model highlight
- Dedicated presentation mode for client-facing model review meetings
- CheckPoint cloud tool for lighter-touch access

**Integration points**
- Java-based Developer Platform API (solibri.github.io) for custom rule authoring
- BCF export/import for issue exchange with BIMcollab, Revit, ArchiCAD, Navisworks
- Autodesk Revit and ArchiCAD plugins for direct model publishing
- Documents API for CDE (common data environment) integration

**Known gaps**
- Steeper learning curve than geometry-only tools
- Desktop software requires local installation (though CheckPoint addresses cloud access)
- Ruleset customisation requires Java development knowledge
- No real-time multi-user collaboration workspace; issue exchange is file-based BCF

**Licence / IP notes**
- Proprietary commercial licence; acquired by Nemetschek Group
- Developer Platform API published openly on GitHub; examples under open licence

---

### Trimble Connect

**Core features**
- Cloud-based 3D model federation and clash detection (desktop + browser)
- Clash sets between loaded models; file-to-TrimBIM conversion for processing
- Supports IFC, TrimBIM, SketchUp, point cloud, and PDF 3D documents
- Issue tracking linked to clash results and model objects
- Integration with Tekla Structures for structural-centric workflows
- MEP fabrication coordination via SysQue integration
- Task management and to-do assignment linked to model objects
- Version management and model comparison

**Differentiating features**
- Best-in-class for Tekla Structures-centric structural workflows
- Clash detection included in all pricing tiers (not a premium add-on)
- SysQue MEP fabrication connection bridges design coordination to prefabrication

**UX patterns**
- Browser-based viewer with overlay of discipline models
- Clash list panel alongside 3D view
- Task cards assigned to model elements

**Integration points**
- Trimble Connect API (REST) for model, issue, and task management
- Tekla Structures direct publish and sync
- SysQue MEP fabrication platform
- IFC open standard for cross-platform model exchange

**Known gaps**
- Clash detection engine not as widely adopted as Navisworks
- Smaller ecosystem of third-party integrations vs. Autodesk/BIMcollab
- Less feature-rich issue workflow customisation

**Licence / IP notes**
- Commercial SaaS; freemium entry tier available
- API documented publicly; REST/JSON standard

---

### Newforma Konekt (formerly BIM Track)

**Core features**
- Clash-to-issue conversion: imports Navisworks clash results as structured tracked issues
- Integration with Revit, ArchiCAD, Navisworks, and AVEVA E3D Design via BCF
- Action item and issue tracking with owner, due date, priority, and status
- Integrated 2D/3D viewer with markup and annotation
- Email and Outlook integration to capture coordination communications
- Real-time sync of issue updates to all stakeholders
- AVEVA E3D Design integration for industrial/process plant coordination

**Differentiating features**
- Only platform with deep AVEVA E3D Design integration — targets industrial, oil and gas, and process plant projects
- Email integration captures informal coordination decisions alongside formal issues
- Bridges design coordination (BIM) to project information management (PIM)

**UX patterns**
- Web-based dashboard with issue list, filter, and map views
- Outlook add-in for capturing emails as coordination issues
- Embedded viewer for issue location context

**Integration points**
- BCF-API for Revit, ArchiCAD, Navisworks
- AVEVA E3D Design BCF integration
- Navisworks add-in for direct clash-to-issue workflow
- REST API for custom integrations

**Known gaps**
- Less established than Navisworks or Revizto in pure BIM coordination contexts
- Advanced features require higher subscription tiers
- Branding transition from BIM Track may still cause market confusion

**Licence / IP notes**
- Commercial SaaS; no open-source components identified

---

### BIMWorkplace

**Core features**
- Centralised cloud model hub for uploading Revit, IFC, and NWC models
- Structured clash issue assignment and tracking workflow
- Dashboard analytics: open vs. closed topics, team workload, bottleneck detection
- Visual issue cards with model screenshots and assignee context
- Integrates with Navisworks for clash input, then manages resolution workflow
- Collaboration tools replacing ad-hoc emails and spreadsheets for coordination

**Differentiating features**
- Explicitly focused on BIM Managers' workflow pain — not a full model viewer but a coordination management layer
- Analytics dashboards for coordination progress and team performance

**UX patterns**
- Kanban-style issue workflow boards
- Progress metrics for open/closed clash topics
- Team workload visibility across disciplines

**Integration points**
- Navisworks for clash input
- IFC model viewer embedded
- Export to standard formats for reporting

**Known gaps**
- Less mature product than established players
- Limited advanced clash configuration; relies on Navisworks for actual detection
- Smaller user community and fewer third-party integrations

**Licence / IP notes**
- Commercial SaaS; pricing not publicly listed

---

### AutoBIMRoute

**Core features**
- AI-driven automated MEP routing and placement that generates clash-free layouts
- Analyses spatial constraints, system priorities, and constructability rules
- Produces coordinated layouts complying with engineering standards
- Clash avoidance by design — prevents conflicts before they are detected
- Integrates with Revit for direct model output

**Differentiating features**
- Clash avoidance (generative) rather than clash detection (diagnostic) — fundamentally different paradigm
- AI routing learns from building geometry and system priorities

**UX patterns**
- Revit plugin interface with configuration panel for routing rules
- Automated generation of coordinated MEP paths

**Integration points**
- Autodesk Revit plugin
- IFC export for cross-platform review

**Known gaps**
- Primarily MEP-focused; does not address structural or architectural coordination
- Still emerging product — limited public documentation on accuracy and scale
- No standalone viewer or issue tracking

**Licence / IP notes**
- Commercial; pricing not publicly disclosed

---

### Genusys.ai

**Core features**
- AI-driven automation for MEP BIM specifically targeting mission-critical construction (data centres, hospitals)
- Automated clash detection and avoidance for electrical and MEP designs
- Clash-free MEP layout generation compatible with Revit
- Auto-routing with conflict detection integrated in generation step
- Top BIM automation tools summary published for 2026

**Differentiating features**
- Mission-critical project specialisation (data centres, hospitals) — niche but high-value
- Combines routing generation with embedded clash avoidance rather than post-generation checking

**UX patterns**
- Revit plugin / cloud processing workflow
- Automated output delivered as coordinated Revit model

**Integration points**
- Autodesk Revit
- IFC export

**Known gaps**
- Niche target market limits breadth of adoption
- No published open API or BCF support identified
- Limited public user reviews or case studies

**Licence / IP notes**
- Commercial SaaS; pricing not publicly disclosed

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Hard clash detection between discipline model pairs (geometric intersection)
- Soft clash detection with configurable clearance tolerances
- Clash results list with status tracking (New, Active, Resolved, Approved)
- Issue assignment with owner, due date, and priority
- 3D viewer with clash highlight and navigation to clash location
- BCF export/import for cross-platform issue exchange
- Clash report generation (PDF, HTML, or CSV)
- User access controls and permissions per project

### Differentiating Features
- 4D clash detection (time-based, construction sequencing) — Navisworks
- Rules-based compliance checking beyond geometry — Solibri
- AI-assisted MEP routing for proactive clash avoidance — AutoBIMRoute, Genusys.ai
- 2D/3D simultaneous issue location context — Revizto
- Auto-status update of Smart Issues on model sync — BIMcollab
- Industrial/AVEVA E3D integration — Newforma Konekt
- Real-time multi-user collaboration workspace — Revizto, ACC
- Reusable clash check configurations across projects — ACC

### Underserved Areas / Opportunities
- AI-driven false-positive filtering to reduce clash noise (thousands of low-priority clashes overwhelm teams)
- Intelligent clash grouping and priority ranking without manual setup
- Natural-language clash query interface ("show me all MEP-structural clashes above Level 3")
- Automated clash matrix setup from model metadata and BIM Execution Plan
- Cross-platform BCF issue aggregation from multiple tools into a single view without vendor lock-in
- Historical clash trend analytics to identify recurring coordination failure patterns
- Open-source clash detection engine for non-Autodesk model pipelines
- Lightweight stakeholder access (no full software licence required) for subcontractors and clients

### AI-Augmentation Candidates
- Clash relevance scoring: ML classifier trained on historical resolution data to filter nuisance clashes
- Automated severity ranking: predict which clashes have highest on-site rework cost
- Clash clustering: group related clashes by root cause (e.g., all clashes from a misaligned grid)
- Resolution suggestion: recommend standard fixes from similar past clash resolutions
- Proactive routing generation: AI-generated MEP routes that avoid structural elements before clash testing
- Natural-language issue creation and assignment from clash context
- Clash trend forecasting: identify discipline pairs or zones likely to generate future clashes from model history

---

## Legal & IP Summary

All major platforms surveyed are proprietary commercial software. The BCF (BIM Collaboration Format) standard and its REST API specification are open standards published by buildingSMART International under open terms, making issue data portable between tools. IFC (ISO 16739-1:2024) is an open ISO standard. No patents were identified covering specific clash detection algorithms or workflows in the public literature. Navisworks NWD/NWC formats are proprietary; tools relying on them for model input depend on Autodesk licence terms. Open-source implementations of IFC parsing (IfcOpenShell, xBIM) are available under LGPL and MIT licences respectively, providing a foundation for building independent clash detection tooling without proprietary format dependency.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Hard and soft clash detection on federated IFC models
- Configurable tolerance and discipline-pair test matrix
- Clash grouping by zone, level, and discipline
- Issue lifecycle management (assign, status, comment, resolve) with BCF export
- Browser-based viewer with clash highlight — no installed software for stakeholders
- Basic clash report generation (PDF/CSV)

**Should-have (v1.1)**
- AI-assisted clash relevance scoring to filter nuisance results
- Reusable clash check configuration templates across projects
- Natural-language query interface for filtering clash results
- Integration with Autodesk ACC, BIMcollab, and Procore via BCF and REST APIs
- Automated re-detection on each model upload with change-diff highlighting
- Dashboard analytics: open/closed trends, team workload, bottleneck detection

**Nice-to-have (backlog)**
- 4D clash detection (time-based sequencing conflicts)
- AI-generated MEP routing suggestions for proactive clash avoidance
- Automated root-cause clustering across large clash sets
- AVEVA E3D Design integration for industrial projects
- Mobile app for field access and on-site issue capture
- Resolution suggestion engine trained on historical project data
