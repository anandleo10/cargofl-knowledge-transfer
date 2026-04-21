# CargoFL Research Agent — Knowledge Transfer

This repository contains the complete Knowledge Transfer (KT) documentation for the **CargoFL Research Agent** — an AI-powered logistics intelligence platform built for the CargoFL / PUMA operations team.

## Contents

| Document | Description |
|----------|-------------|
| [00_Overview.md](00_Overview.md) | Product overview, capability map, UI walkthrough |
| [01_TechStack_Architecture.md](01_TechStack_Architecture.md) | Full tech stack, system architecture diagram |
| [02_FolderStructure.md](02_FolderStructure.md) | Every folder and file annotated |
| [03_DataFlow_Workflows.md](03_DataFlow_Workflows.md) | End-to-end workflows: query pipeline, Maria cycle, Q&A |
| [04_APIReference.md](04_APIReference.md) | All ~60 REST API endpoints |
| [05_ControlTower.md](05_ControlTower.md) | 5 CT views, cache mechanics, nightly refresh |
| [06_MariaAgent.md](06_MariaAgent.md) | Maria agent deep dive — jobs, alerts, Q&A, thresholds |
| [07_Frontend.md](07_Frontend.md) | React architecture, 4 tabs, components, api.ts |
| [08_DeploymentOps.md](08_DeploymentOps.md) | Docker setup, deploy procedures, monitoring |
| [09_Configuration.md](09_Configuration.md) | Every config variable and file explained |
| [10_Diagrams/](10_Diagrams/) | Mermaid diagrams: system, query flow, Maria flow, CT, Docker |
| [GLOSSARY.md](GLOSSARY.md) | Domain terms, code terms, abbreviations |

## Recommended Reading Order

**Day 1 — Big Picture**
→ `00_Overview` → `01_TechStack_Architecture` → `10_Diagrams/system_architecture`

**Day 2 — Code & Data Flow**
→ `02_FolderStructure` → `03_DataFlow_Workflows` → `10_Diagrams/query_workflow`

**Day 3 — Features**
→ `05_ControlTower` → `06_MariaAgent` → `10_Diagrams/maria_workflow`

**Day 4 — Ops & Config**
→ `07_Frontend` → `08_DeploymentOps` → `09_Configuration`

**Reference**
→ `04_APIReference` → `GLOSSARY`

---

*Generated April 2026 — CargoFL / Innoctive*
