# BAITEKS Copilot Memory Layer

> Reusable SQL/Dataverse-backed persistent memory architecture for Microsoft Copilot Studio agents.

---

## The Problem

Microsoft Copilot Studio has **no persistent memory**. Every conversation starts from zero. The bot doesn't know who it's talking to, what they asked yesterday, or what institutional knowledge exists about the equipment they're asking about.

In industrial environments — refineries, chemical plants, power generation — this is not a minor gap. Planners, operators, and engineers carry decades of equipment-specific knowledge. When they retire, that knowledge leaves with them. A bot with amnesia can't capture or surface any of it.

---

## The Two-Layer Architecture

The Copilot Memory Layer solves this with two complementary persistence layers that bolt onto any Copilot Studio agent:

### Layer 1 — User Memory

Per-user personalization that makes the bot immediately useful:

- **UserProfile** — role, craft, department, unit assignments, resolved via M365 Azure AD
- **UserQueryHistory** — automatic logging of every question asked, with timestamps and topic categories
- **UserSavedQueries** — bookmarked queries users run repeatedly, ranked by use count
- **UserPreferences** — key-value pairs for equipment filters, display settings, and workflow preferences

At session start, a Power Automate flow loads the user's profile, recent history, and top saved queries into the system prompt. The bot knows who it's talking to before the user types a word.

### Layer 2 — Plant Memory

Institutional knowledge that never leaves the organization:

- **PlantLessonsLearned** — structured lessons keyed to equipment class, tag, and failure mode, with verification workflow
- **ApprovedProcedures** — step-by-step procedures, versioned and approved, stored as structured JSON
- **EquipmentNotes** — per-equipment tribal knowledge: tips, warnings, and historical context
- **PlantGlossary** — standardized plant terminology so the bot speaks the same language as the crew

When a user asks about a piece of equipment, Plant Memory is searched and the relevant lessons, notes, and warnings are injected directly into the response — cited, dated, and attributed.

---

## How It Works

```
┌─────────────────────────────────────────────────────┐
│                 Copilot Studio Agent                 │
│            (LIAM, P6 Intelligence, etc.)            │
├──────────────────────┬──────────────────────────────┤
│    System Prompt     │     Actions (read/write)     │
│    Injection         │                              │
├──────────────────────┴──────────────────────────────┤
│              Power Automate Flows                   │
│   ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│   │ Load User│ │Log Query │ │ Get Plant Memory  │  │
│   │ Context  │ │          │ │                   │  │
│   └──────────┘ └──────────┘ └───────────────────┘  │
├─────────────────────────────────────────────────────┤
│           SQL Server / Dataverse                    │
│  ┌─────────────────┐  ┌──────────────────────────┐ │
│  │  USER MEMORY     │  │  PLANT MEMORY            │ │
│  │  · UserProfile   │  │  · PlantLessonsLearned   │ │
│  │  · QueryHistory  │  │  · ApprovedProcedures    │ │
│  │  · SavedQueries  │  │  · PlantGlossary         │ │
│  │  · Preferences   │  │  · EquipmentNotes        │ │
│  └─────────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## Wiki Pages

| Page | Description |
|------|-------------|
| **[Database Schema](Database-Schema)** | Complete SQL Server CREATE TABLE statements for all 8 tables, with indexes, foreign keys, and Dataverse mapping |
| **[Power Automate Flows](Power-Automate-Flows)** | Step-by-step specs for all 5 flows: session start, query logging, plant memory retrieval, lesson capture, and saved queries |
| **[Plant Memory Guide](Plant-Memory-Guide)** | Deep dive on institutional knowledge capture — the retirement crisis, what to capture, how to build the culture, and real examples |

---

## Fits Into

This is a **building block, not a standalone app.** It bolts onto any Copilot Studio agent:

- **LIAM** — turnaround planning assistant
- **P6 Intelligence** — Primavera P6 copilot
- **Future SAP Intelligence** — SAP copilot for maintenance and materials
- **Any enterprise Copilot Studio bot** that benefits from knowing its users and retaining institutional knowledge

One memory layer, many agents.

---

*Built by [BAITEKS](https://baiteks.com)*
