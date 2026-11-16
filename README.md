# migration-agent

> **Claude-powered migration agent — moves content from legacy enterprise systems into Microsoft 365, with Claude as the reasoning engine and Model Context Protocol (MCP) as the integration layer.**

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Status: Design](https://img.shields.io/badge/Status-Design-orange.svg)](#project-status)
[![Python 3.11+](https://img.shields.io/badge/Python-3.11+-green.svg)](https://www.python.org/)

---

## Why this exists

Enterprise content migration is **expensive, error-prone and slow**. Typical consultancy-led migrations cost £5k–£50k per TB, take months per wave, and require rule-writing that breaks on edge cases nobody saw in discovery.

Most existing tools (ShareGate, Proventeq Migration Accelerator, Quest Content Matrix, SPMT, AvePoint DocAve) handle the mechanics well but ask a human to write every mapping decision, every exception rule, every permission remediation strategy. That human is normally a £700/day consultant working through thousands of edge cases.

**Claude reasons about exceptions.** Given a source item it has never seen, Claude can decide what content type it should become, how its metadata should map, whether its permissions need remediation, and whether it should be skipped, flagged, or migrated as-is — with explanations that a human reviewer can audit.

`migration-agent` wraps Claude as the planning and decision engine around a conventional migration execution pipeline, connected to Microsoft 365 through an open Model Context Protocol server. The result: faster migrations, better decisions on edge cases, auditable reasoning, and far less consultant time per TB.

## What it does

Given a source content system and a Microsoft 365 target, `migration-agent`:

1. **Discovers** the source — folder structure, file types, metadata, permissions, sizes, ownership
2. **Plans** the migration — target site structure, content type mapping, permission strategy, wave sequencing (Claude as planner)
3. **Previews** in dry-run mode with no destructive operations and full exception list
4. **Executes** with resume-on-failure, full audit logging and human-in-the-loop wave gates
5. **Validates** post-migration — content counts, metadata integrity, permission equivalence, broken-link detection
6. **Reports** — execution logs, metrics, exception register, recommendations for cleanup

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  CLI / Web UI / API                                 │
└─────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  Migration Agent (Python)                           │
│  ─ Anthropic SDK                                    │
│  ─ Claude as planner, decision-maker, explainer     │
│  ─ State store (SQLite) for resume + audit          │
│  ─ Wave orchestrator                                │
└─────────────────────────────────────────────────────┘
                     │                    │
       Source────────┤                    ├────────Target
                     ▼                    ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│  Source Adapter          │   │  mcp-sharepoint          │
│  (dms-adapters repo)     │   │  (MCP server for SPO)    │
│  ─ Filesystem            │   │  ─ Create sites/libraries│
│  ─ OpenText / Livelink   │   │  ─ Upload files          │
│  ─ iManage               │   │  ─ Set metadata          │
│  ─ FileNet               │   │  ─ Apply permissions     │
│  ─ NetDocs / M-Files     │   │  ─ Query / validate      │
└──────────────────────────┘   └──────────────────────────┘
                                          │
                                          ▼
                                 ┌──────────────────────┐
                                 │  SharePoint Online   │
                                 │  (via Microsoft      │
                                 │   Graph API)         │
                                 └──────────────────────┘
```

## Design principles

1. **Dry-run first.** No destructive operation runs without a preview mode first. Every wave produces an execution plan the user can inspect before a single byte is copied.
2. **Resumable.** State persisted at every checkpoint. Crash, network loss, or user interruption never costs completed work. Restart picks up where it left off.
3. **Auditable.** Every decision Claude makes is logged with its reasoning. Every API call is logged with request, response and timing. Full chain of custody for every migrated item.
4. **Human-in-the-loop.** Claude recommends and plans. Humans approve wave gates, resolve ambiguous mappings, and sign off on destructive actions. The agent never makes irreversible decisions alone.
5. **Extensible.** Adding a new source system is writing one adapter against a documented interface. The core agent stays unchanged.
6. **Open standards.** Target integration goes through Model Context Protocol, not vendor-proprietary APIs. Adapters are open source. Reasoning layer is swappable.
7. **Metadata-preserving.** Modified dates, created dates, authors, versions, permissions remain intact — users returning to the migrated content see what they expect to see.
8. **Secure by default.** Source credentials never leave the local environment. Content streams through the agent rather than being staged in intermediate storage. Audit logs retain only metadata by default, not content.

## Technology stack

- **Python 3.11+** — primary implementation language
- **Anthropic SDK** — Claude reasoning, tool use, agentic patterns
- **MCP Python SDK** — Model Context Protocol client to `mcp-sharepoint`
- **Microsoft Graph SDK** — SharePoint Online operations (indirect, via MCP server)
- **SQLite** — local state store, audit log, resume checkpoints
- **Typer** — CLI framework
- **Rich** — terminal UI, progress bars, tables
- **pytest** — test framework
- **GitHub Actions** — CI/CD

## Project status

**Current phase**: Design and architecture. This repository holds the architectural specification; implementation is phased per the roadmap below.

### Roadmap

| Phase | Period | Deliverable |
|---|---|---|
| **Phase 0** | Nov–Dec 2026 | Architecture specification, project README, design decisions |
| **Phase 1** | Jan–Feb 2027 | `mcp-sharepoint` server v0.1 (prerequisite — separate repository) |
| **Phase 2** | Mar–May 2027 | `migration-agent` MVP — file share → SharePoint Online migration |
| **Phase 3** | Jun–Aug 2027 | OpenText Content Server / Livelink adapter |
| **Phase 4** | Sep–Nov 2027 | iManage Work adapter |
| **Phase 5** | 2028 Q1 | IBM FileNet adapter |
| **Phase 6** | 2028 Q2 | NetDocs, M-Files adapters |
| **Phase 7** | 2028+ | Web UI, scheduling, multi-tenant support, enterprise features |

### Supported source systems

| Source | Status | Credential anchor |
|---|---|---|
| Local / network file shares (SMB / UNC) | 🎯 MVP target | Multiple engagements |
| OpenText Content Server / Livelink | 📋 Planned Phase 3 | Lincolnshire County Council (14 TB), Vattenfall |
| iManage Work | 📋 Planned Phase 4 | Arthur Cox LLP (18 TB) |
| IBM FileNet P8 | 📋 Planned Phase 5 | JM&A Group |
| NetDocuments | 📋 Planned Phase 6 | Growing in legal sector |
| M-Files | 📋 Planned Phase 6 | Growing in regulated industries |
| Box, Documentum, Hummingbird | 📋 Future consideration | Based on demand |

## Getting started

> **Note**: implementation has not yet started. Expected first release Q2 2027.

When available, installation will be:

```bash
pip install migration-agent
migration-agent --help
```

Watch and star this repository to be notified when the first release lands.

## Scope

**In scope**:
- Content migration — files, documents, metadata, folder structure
- Permission preservation and remediation
- Content type mapping and taxonomy alignment
- Version history (where source supports it)
- Links between items (where resolvable)
- Audit trail of all actions and Claude reasoning

**Out of scope**:
- Live two-way synchronisation (different product category)
- Active workflow or form migration (handle through separate transformation projects)
- eDiscovery export (adjacent tooling — may integrate but not reimplement)
- Email migration (BitTitan, SkyKick and similar already solve this well)

## Security and compliance considerations

- **Authentication**: source and target credentials stay local; never transmitted to Anthropic
- **Content handling**: file content is streamed between source and target; Claude receives metadata and summaries, not full content by default (configurable per use case)
- **Audit retention**: logs retain actions, decisions, and reasoning for configurable retention periods
- **Data residency**: all file bytes remain within the customer environment unless Claude content analysis is explicitly enabled per-wave
- **Encryption**: TLS in transit, SQLite state store encrypted at rest via platform keyring
- **Compliance alignment**: designed with GDPR, UK Data Protection Act, and common enterprise information governance frameworks in mind

Full threat model and security architecture document to follow before MVP release.

## Contributing

Contributions welcome once the implementation phase begins. Until then:

- **Design feedback** — open an issue with suggestions, concerns, or scenarios the architecture should address
- **Use case input** — tell us about migrations you have run, pain points, edge cases that broke existing tools
- **Source system interest** — vote up planned adapters, suggest additional ones

Full contribution guidelines (`CONTRIBUTING.md`) to follow at MVP release.

## Licence

This project is licensed under the **Apache License 2.0** — see [LICENSE](./LICENSE) for details.

## Maintainer

Built by **[Anika Tech Solution](https://github.com/anika-tech-solution-ltd)** — specialist Microsoft 365, SharePoint and Anthropic Claude consultancy.

Lead: **Arvind Gupta** — Principal Consultant. 15+ years enterprise SharePoint and M365 delivery. Microsoft Certified (AI-103, AB-650, MS-900, PL-900). Anthropic Partner Program applicant.

Current engagement credentials include 60+ TB cumulative enterprise content migrations across legal, public sector, Big 4, energy and global NGO environments. Full delivery profile at [LinkedIn](https://linkedin.com/in/arvind-gupta-bba3bb2).

**Website**: [anikatechsolution.co.uk](https://anikatechsolution.co.uk)

---

*This repository is part of the Anika Tech Solution open-source toolkit for AI-powered enterprise content operations. See the [organisation profile](https://github.com/anika-tech-solution-ltd) for the full portfolio.*
