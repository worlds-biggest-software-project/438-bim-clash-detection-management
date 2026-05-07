# BIM Clash Detection & Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for federating BIM models across disciplines, detecting spatial clashes automatically, and managing the full resolution lifecycle -- eliminating costly on-site rework before construction begins.

BIM Clash Detection & Management brings together architectural, structural, and MEP models from any authoring tool into a single coordinated environment, runs configurable hard and soft clash tests, and tracks every issue from detection through resolution. It is built for BIM coordinators, design leads, and project managers who need transparent, accessible coordination without per-seat licence lock-in.

---

## Why BIM Clash Detection & Management?

- **Vendor lock-in dominates the market.** Autodesk Navisworks Manage is the industry standard, but it uses proprietary NWD/NWC formats and has no native cloud collaboration. Teams are forced into the Autodesk Construction Cloud ecosystem at premium pricing (BIM Collaborate Pro) for browser-based access.
- **Clash noise overwhelms coordination teams.** Large projects generate thousands of clashes, many of which are false positives or low priority. No incumbent offers built-in AI-assisted filtering or relevance scoring -- teams waste hours triaging nuisance results manually.
- **Non-BIM stakeholders are locked out.** Subcontractors, clients, and field teams often cannot access or interpret clash results without expensive desktop software licences. Lightweight browser-based access remains a premium-tier feature.
- **Cross-platform coordination is fragmented.** Projects using mixed toolchains (Revit, ArchiCAD, Tekla, AVEVA) must rely on BCF file exchange between siloed tools. No open-source platform aggregates clash issues from multiple sources into a single view.
- **AI-native clash avoidance is emerging but proprietary.** Tools like AutoBIMRoute and Genusys.ai offer AI-driven MEP routing, but they are closed-source, narrowly focused, and priced for enterprise buyers only.

---

## Key Features

### Model Federation & Clash Detection

- Federate discipline-specific models from Revit, ArchiCAD, Tekla, and other authoring tools via IFC
- Hard clash detection (physical intersection) and soft clash detection (configurable clearance tolerances)
- Configurable discipline-pair test matrix with tolerance settings and rule-based filtering
- Clash grouping by zone, level, discipline, and custom properties
- Automated re-detection on each model upload with change-diff highlighting

### Issue Lifecycle Management

- Convert clash results into tracked issues with owner, due date, priority, and status
- Full issue lifecycle: New, Active, Reviewed, Approved, Resolved
- Comment history, visual snapshots, and model-linked location context
- BCF (BIM Collaboration Format) export/import for cross-platform issue exchange
- Reusable clash check configuration templates across projects

### Browser-Based Viewer & Collaboration

- 3D model viewer with clash highlight and navigation -- no installed software required
- Lightweight stakeholder access for subcontractors, clients, and field teams
- Real-time status updates visible to all project participants
- Role-based access controls and permissions per project

### Reporting & Analytics

- Clash report generation in PDF and CSV formats
- Dashboard analytics: open/closed trends, team workload, bottleneck detection
- Historical clash trend analytics to identify recurring coordination failure patterns

### AI-Augmented Coordination

- AI-driven clash relevance scoring to filter nuisance results and reduce noise
- Automated severity ranking predicting highest on-site rework cost
- Clash clustering by root cause (e.g., all clashes from a misaligned grid)
- Natural-language query interface for filtering results ("show me all MEP-structural clashes above Level 3")
- Resolution suggestions drawn from similar past clash resolutions

---

## AI-Native Advantage

Current market leaders treat clash detection as a purely geometric operation, leaving teams to manually sift through thousands of results. An AI-native approach changes the workflow fundamentally: ML classifiers trained on historical resolution data score each clash for relevance and severity, automatically filtering out false positives that waste coordination time. Natural-language querying lets coordinators interrogate clash data without constructing complex filter chains. Over time, clash trend forecasting identifies discipline pairs and building zones likely to generate future conflicts, enabling proactive intervention rather than reactive detection.

---

## Tech Stack & Deployment

- **Open standards first:** IFC (ISO 16739-1:2024) for model ingestion; BCF (buildingSMART open standard) for issue exchange with any compatible tool
- **Open-source foundations:** IFC parsing via IfcOpenShell (LGPL) or xBIM (MIT), avoiding dependency on proprietary Autodesk formats
- **Deployment modes:** Self-hosted for firms with data residency requirements; cloud-hosted for teams wanting zero-infrastructure coordination
- **Integration targets:** Autodesk ACC, BIMcollab, Procore, and Trimble Connect via BCF and REST APIs
- **Extensibility:** Plugin architecture for authoring tool connectors (Revit, ArchiCAD, Tekla, AVEVA E3D Design)

---

## Market Context

The BIM coordination and clash detection market is dominated by Autodesk Navisworks and ACC, with Solibri, BIMcollab, and Revizto as significant competitors. Pricing models are per-seat subscriptions or platform fees that scale steeply for larger teams -- Revizto does not publicly list pricing, and ACC requires the BIM Collaborate Pro tier for clash detection. Primary buyers are BIM coordinators, MEP contractors, and large design firms managing multi-discipline hospital, airport, data centre, and commercial tower projects, where a single undetected clash can cost tens of thousands of dollars in construction rework.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
