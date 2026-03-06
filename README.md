# copilot-memory-layer

> Reusable SQL/Dataverse-backed persistent memory architecture for Microsoft Copilot Studio agents. Two layers: **User Memory** (per-user personalization) + **Plant Memory** (institutional knowledge that never leaves).

[![Built by BAITEKS](https://img.shields.io/badge/Built%20by-BAITEKS-0066cc)](https://baiteks.com)
[![Status](https://img.shields.io/badge/Status-Active%20Development-brightgreen)]()

---

## The Problem

Microsoft Copilot Studio has no persistent memory. Every session starts from zero. Your bot doesn't know who it's talking to, what they asked yesterday, or what matters to them.

Now multiply that by the real crisis: **your best planner retires and takes 30 years of institutional knowledge with them.** They knew which pump fails every spring, which vendor procedure actually works, which P6 logic was set up wrong in 2014 and why. That knowledge lived in their head. Now it's gone.

In industrial environments — refineries, chemical plants, power generation — this isn't an inconvenience. It's catastrophic. Planners carry years of context about specific equipment, approved workarounds, and lessons learned the hard way. A bot with amnesia can't retain any of it.

**The cost:** longer turnarounds, repeated mistakes, slower onboarding, and decisions made without context that used to be a phone call away.

---

## The Solution

**Two-layer persistent memory** that bolts onto any Copilot Studio agent:

**User Memory** — per-user personalization that makes the bot actually useful:
- Role, craft, unit assignments
- Saved queries they run repeatedly
- Recent query history for continuity
- Equipment and filter preferences

**Plant Memory** — institutional knowledge that never leaves the organization:
- Lessons learned from past turnarounds and incidents
- Approved procedures and verified workarounds
- Equipment-specific notes (the tribal knowledge)
- Plant glossary and terminology

**Captured once, available forever.** When a user opens Copilot, their context loads automatically. When anyone asks about a piece of equipment, decades of accumulated knowledge surfaces in the response — not buried in a SharePoint folder nobody reads.

---

## Features

- **User profiles** — role, craft, unit, preferences stored per user
- **Query history** — automatic logging of what users ask, with timestamps
- **Saved queries** — users bookmark their most-used queries for one-click access
- **Lessons learned** — structured capture of institutional knowledge with verification status
- **Equipment notes** — per-equipment tribal knowledge, searchable by tag
- **Plant glossary** — standardized terminology so the bot speaks the plant's language
- **Verified procedures** — approved workarounds and methods, with audit trail
- **System prompt injection** — user context loaded into the system prompt at session start
- **Session start context loading** — Power Automate fires on conversation open, bot is ready before the user types

---

## Architecture

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

## Tech Stack

| Component | Technology | Purpose |
|---|---|---|
| Backend | SQL Server / Dataverse | Persistent memory storage |
| Automation | Power Automate | Read/write flows between agent and database |
| Agent Platform | Microsoft Copilot Studio | Bot framework |
| Identity | Microsoft 365 | User identification via Azure AD |
| Context Injection | System prompt variables | Session personalization at conversation start |

---

## How It Works

1. **User opens Copilot** — conversation begins, identity resolved via M365.
2. **Power Automate fires** — loads user profile, recent queries, saved queries, and preferences from SQL/Dataverse.
3. **Context injected into system prompt** — formatted block inserted so the bot knows who it's talking to, what they care about, and what they asked last time.
4. **Bot responds with full context** — no "who are you?" no "what department?" It already knows.
5. **User asks about equipment** — Plant Memory searched by equipment tag, unit, or keyword.
6. **Lessons and notes surface** — institutional knowledge injected directly into the response, cited and verified.
7. **Queries logged automatically** — history builds over time, improving personalization.

---

## Plant Memory — The Standout Feature

Industry is facing a 30-50% retirement wave over the next decade. When a senior planner, operator, or engineer walks out the door, they take knowledge that took decades to accumulate. The workaround for the valve that sticks in cold weather. The vendor procedure that sounds right but causes problems on Unit 4. The reason a specific logic tie exists in the P6 schedule.

**Plant Memory captures this in a living, searchable database** — not a static document, not a SharePoint library that gets stale, not an email chain someone might find. It's structured, verified, and injected into AI responses exactly when it's relevant. The knowledge stays with the plant, not the person.

---

## Fits Into

This is a **building block, not a standalone app.** It bolts onto any Copilot Studio agent:

- **LIAM** — turnaround planning assistant
- **P6 Intelligence** — Primavera P6 copilot
- **Future SAP Intelligence** — SAP copilot for maintenance and materials
- **Any enterprise Copilot Studio bot** that benefits from knowing its users and retaining institutional knowledge

One memory layer, many agents.

---

## Documentation

Full architecture details, SQL schema, Power Automate flow specs, and integration guide:

- [docs/COPILOT-MEMORY-LAYER.md](docs/COPILOT-MEMORY-LAYER.md) — Complete technical documentation

---

## Roadmap

- [ ] **Phase 1:** SQL schema + User Memory tables + session start flow
- [ ] **Phase 2:** Plant Memory tables + lesson capture + retrieval flows
- [ ] **Phase 3:** Copilot Studio actions + system prompt injection
- [ ] **Phase 4:** Admin view + verification workflow + Dataverse migration option
- [ ] **Phase 5:** Analytics dashboard + cross-agent memory sharing

---

## License

MIT
