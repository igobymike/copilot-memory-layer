# Copilot Memory Layer — Architecture Document

**Version:** 1.0
**Author:** BAITEKS Engineering
**Date:** 2026-03-06
**Status:** Design Specification
**Classification:** Internal — Technical Architecture

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Memory Problem in Copilot Studio](#2-the-memory-problem-in-copilot-studio)
3. [Two-Layer Architecture](#3-two-layer-architecture)
4. [Full Database Schema](#4-full-database-schema)
5. [Power Automate Flows](#5-power-automate-flows)
6. [Copilot Studio Integration](#6-copilot-studio-integration)
7. [Dataverse Option](#7-dataverse-option)
8. [Privacy & Access Control](#8-privacy--access-control)
9. [LIAM + P6 Intelligence Integration Examples](#9-liam--p6-intelligence-integration-examples)
10. [Build Roadmap](#10-build-roadmap)
11. [The Big Picture — Why This Matters](#11-the-big-picture--why-this-matters)

---

## 1. Executive Summary

Microsoft Copilot Studio agents have no persistent memory. When a user closes a conversation, every piece of context — who they are, what they asked, what the bot figured out — is gone. The next session starts from zero. For a consumer chatbot answering FAQ questions, this is tolerable. For an industrial AI assistant that a maintenance planner uses daily to manage thousands of work orders, equipment procedures, and scheduling decisions, it is unacceptable.

The **Copilot Memory Layer** solves this with a reusable, SQL/Dataverse-backed persistent memory architecture that bolts onto any Copilot Studio agent. It has two layers:

- **Layer 1 — User Memory:** Per-user personalization and continuity. The bot remembers who you are, what you work on, what you asked last week, and what queries you have saved. When Mike Birklett opens LIAM on Monday morning, the bot already knows he is a maintenance planner, he works on pumps and heat exchangers, and his daily summary query is ready to run.

- **Layer 2 — Plant Memory:** Institutional knowledge shared across all users. Lessons learned, equipment-specific notes, approved procedures, and plant glossary terms. When any planner asks about a specific piece of equipment, relevant knowledge from the entire organization surfaces automatically. This is not a document library — it is a living, queryable knowledge base that grows richer with every interaction.

This architecture is designed as a **reusable module** within the BAITEKS ecosystem. It is not LIAM-specific or P6 Intelligence-specific. Any Copilot Studio agent — current or future — can plug into the same memory layer. The schema, Power Automate flows, and Copilot Studio actions are built to be portable.

The stakes are real. The industrial workforce is aging. In petrochemical plants, refineries, and manufacturing facilities, 30-50% of experienced planners, schedulers, and operators will retire within the next decade. When a planner with 25 years of institutional knowledge about specific equipment walks out the door, that knowledge goes with them — unless it has been systematically captured. Plant Memory is the mechanism for that capture.

---

## 2. The Memory Problem in Copilot Studio

### Default Behavior: Amnesia by Design

Copilot Studio agents operate with **session-scoped memory only**. Each conversation exists in isolation. The bot has access to:

- The current conversation transcript (messages exchanged in this session)
- Any topic variables defined within the current flow
- System variables like `User.DisplayName` from the authentication provider

When the session ends — user closes the browser tab, times out, or navigates away — all of that context is destroyed. The next time the same user opens the same bot, the bot has no record of the previous interaction. No memory of preferences. No memory of what equipment they care about. No memory of the lesson learned they mentioned yesterday.

This is the default. Microsoft does not provide a built-in persistent memory layer for Copilot Studio.

### Three Levels of Memory in Copilot Studio

Understanding the problem requires understanding what Copilot Studio actually offers:

**Level 1 — In-Session Context (Built-in)**
The conversation transcript and any variables set during the current topic flow. This is automatic. It works well within a single session. The bot can reference earlier messages, track slot-filling progress, and maintain conversational coherence. But it all evaporates when the session ends.

**Level 2 — Variable Persistence (Limited)**
Copilot Studio allows you to mark certain global variables as "persistent," but this is tied to the authentication session and the specific channel. The persistence is fragile — it depends on how the user connects (Teams vs. web vs. embedded), whether the session token survives, and whether the underlying infrastructure decides to clear state. In practice, you cannot rely on this for anything beyond the current day. It is also limited to simple key-value pairs. You cannot persist structured data, query histories, or complex preferences through this mechanism.

**Level 3 — External Storage (What We Build)**
The only reliable way to give a Copilot Studio agent persistent memory is to write data to an external store (SQL Server, Dataverse, SharePoint, Cosmos DB) and read it back at the start of each session. This is what the Copilot Memory Layer does. It is explicit, structured, reliable, and under our control. We write what matters. We read it back when we need it. We own the schema, the retention policy, and the access model.

### Why This Matters in Industrial Environments

A consumer chatbot might ask "How can I help you today?" every time and that is fine. An industrial AI assistant cannot afford that level of ignorance.

Consider the daily workflow of a maintenance planner at a petrochemical plant:

- They manage 200-500 active work orders across dozens of equipment classes
- They have deep knowledge of specific equipment — which pumps are problematic, which heat exchangers have non-standard gaskets, which compressors have a history of bearing failures
- They have preferred workflows — they always start the day with a backlog summary, then move to scheduling, then handle new notifications
- They have built up mental models over years — "when the seal on P-4401A fails, it is usually the carbon face, not the O-ring, and you need the oversized seat because the bore was machined in 2019"

None of this knowledge exists in the CMMS. It lives in the planner's head, on sticky notes, in personal spreadsheets, and in the muscle memory of daily practice. An AI assistant that cannot learn and retain any of this is an AI assistant that stays at the surface level — useful for basic lookups, but never becoming the trusted tool that a planner actually relies on.

### The Institutional Knowledge Crisis

This is not hypothetical. The numbers are well-documented:

- The American Petroleum Institute estimates that 50% of skilled workers in the oil and gas industry will be eligible for retirement by 2030
- The Bureau of Labor Statistics projects a 15-20% decline in experienced maintenance and operations staff across heavy industry
- A single refinery turnaround may involve 50,000+ activities, with critical path decisions depending on institutional knowledge that exists nowhere in written form

When a senior planner retires, the replacement does not just lose a person — they lose decades of accumulated knowledge about:

- Equipment-specific failure patterns and workarounds
- Vendor preferences and part substitution knowledge
- Informal procedures that work better than the official ones
- Plant-specific terminology and shorthand
- Historical context for why things are done a certain way

Plant Memory is the systematic answer to this problem. Not a document dump. Not a SharePoint graveyard. A living, queryable, growing knowledge base that captures institutional knowledge at the moment it surfaces — during the daily work of planning and maintaining equipment — and makes it available to everyone, forever.

---

## 3. Two-Layer Architecture

### Layer 1 — User Memory

**Purpose:** Personalization, continuity, and saved context. The bot knows who you are and what you care about.

User Memory is personal to each user. It stores profile information, query history, saved queries, and flexible preferences. It is read at session start and written to during and after each session.

#### Tables

**`UserProfile`** — Core user identity and summary data

| Column | Type | Description |
|--------|------|-------------|
| UserID | NVARCHAR(128) | Primary key. Azure AD / M365 Object ID from authentication. |
| DisplayName | NVARCHAR(256) | Full name as returned by Graph API. |
| Email | NVARCHAR(256) | Primary email address. |
| Role | NVARCHAR(128) | Job role — Maintenance Planner, Scheduler, Reliability Engineer, etc. |
| Department | NVARCHAR(128) | Department or unit — VN-PM, Turnaround, Reliability, etc. |
| Plant | NVARCHAR(128) | Plant or facility code. |
| PreferredEquipmentClasses | NVARCHAR(512) | Comma-separated list of equipment classes the user most frequently works with. |
| FrequentTopics | NVARCHAR(512) | Auto-derived from query history — top 5 topic categories. |
| OnboardedDate | DATETIME | When the user first interacted with the system. |
| LastActiveDate | DATETIME | Last time the user had a session. |
| SessionCount | INT | Total number of sessions. |
| IsActive | BIT | Soft delete flag. |

**`UserQueryHistory`** — Every question the user asks, logged silently

| Column | Type | Description |
|--------|------|-------------|
| QueryID | INT IDENTITY | Primary key. |
| UserID | NVARCHAR(128) | Foreign key to UserProfile. |
| Timestamp | DATETIME | When the query was issued. |
| QuestionText | NVARCHAR(2000) | The full text of the user's question. |
| TopicCategory | NVARCHAR(128) | Auto-classified category: Job Plan, Backlog, Scheduling, Equipment Lookup, etc. |
| EquipmentReferenced | NVARCHAR(256) | Equipment tag(s) mentioned in the query, if any. |
| AgentName | NVARCHAR(128) | Which Copilot agent handled this — LIAM, P6 Intelligence, etc. |
| WasHelpful | BIT | Optional user rating. NULL if not rated. |
| ResponseSummary | NVARCHAR(1000) | Brief summary of what the bot returned. For analytics, not replay. |

**`UserSavedQueries`** — User-bookmarked queries they want to reuse

| Column | Type | Description |
|--------|------|-------------|
| SavedQueryID | INT IDENTITY | Primary key. |
| UserID | NVARCHAR(128) | Foreign key to UserProfile. |
| QueryName | NVARCHAR(256) | User-provided name — "VN-PM Daily Summary", "Open Pump WOs", etc. |
| QueryText | NVARCHAR(2000) | The full query text to re-execute. |
| AgentName | NVARCHAR(128) | Which agent this query is for. |
| CreatedDate | DATETIME | When saved. |
| LastUsedDate | DATETIME | When last run. |
| UseCount | INT | How many times executed. Used for ranking. |
| IsActive | BIT | Soft delete flag. |

**`UserPreferences`** — Flexible key-value store for anything else

| Column | Type | Description |
|--------|------|-------------|
| PreferenceID | INT IDENTITY | Primary key. |
| UserID | NVARCHAR(128) | Foreign key to UserProfile. |
| PreferenceName | NVARCHAR(256) | Key — "DefaultView", "TimeZone", "SummaryFormat", etc. |
| PreferenceValue | NVARCHAR(1000) | Value — "compact", "CST", "bullets", etc. |
| UpdatedDate | DATETIME | When last changed. |

#### What Gets Stored Automatically vs. User Action

**Automatic (no user action required):**
- UserProfile creation on first interaction (populated from M365 authentication token)
- LastActiveDate and SessionCount updates on every session start
- QueryHistory entry on every bot response (silent background write)
- FrequentTopics recalculation (periodic, based on QueryHistory aggregation)
- PreferredEquipmentClasses derivation (from EquipmentReferenced in QueryHistory)

**User-initiated:**
- Saved queries — user explicitly says "save this query" or "bookmark this"
- WasHelpful rating — user gives thumbs up/down or explicit feedback
- Role and Department correction — user can update if auto-detected values are wrong
- Preference settings — user explicitly sets display preferences

#### Context Retrieval at Session Start

When a user opens any Copilot Studio agent that uses the memory layer, a Power Automate flow fires (triggered by conversation start). The flow:

1. Authenticates the user via M365 token, extracts UserID
2. Queries `UserProfile` for identity and preferences
3. Queries `UserQueryHistory` for the last 5 queries (most recent, across all agents)
4. Queries `UserSavedQueries` for top 5 by UseCount (where IsActive = 1)
5. Formats a context string
6. Returns the context string to Copilot Studio, which injects it into the system prompt

**Example:** When Mike Birklett opens LIAM, the bot receives this context before the first user message:

```
[USER CONTEXT]
Name: Mike Birklett
Role: Maintenance Planner | Department: VN-PM | Plant: Valero Norco
Top equipment classes: Pumps, Heat Exchangers, Compressors
Session count: 147 | Last active: 2026-03-05

Recent queries (last 7 days):
  1. "pump mechanical seal job plan for P-4401A" (2026-03-05)
  2. "VN-PM backlog summary" (2026-03-05)
  3. "exchanger bundle removal steps HX-4401" (2026-03-04)
  4. "compressor bearing failure history K-2201" (2026-03-03)
  5. "open notifications for VN-PM this week" (2026-03-03)

Saved queries (by use count):
  1. "VN-PM Daily Summary" (used 89 times)
  2. "Pump Seal Job Plan Template" (used 34 times)
  3. "Open HX Work Orders" (used 28 times)
  4. "Weekly Backlog by Priority" (used 22 times)
  5. "Compressor PM Schedule" (used 15 times)

Preferences:
  SummaryFormat: bullets
  DefaultTimeRange: 7days
[/USER CONTEXT]
```

This context allows the bot to greet Mike appropriately, suggest relevant queries proactively, and understand the domain context of his questions without him having to re-establish it every session.

---

### Layer 2 — Plant Memory (Institutional Knowledge)

**Purpose:** Capture lessons learned, approved procedures, equipment-specific notes, and plant terminology. Shared across all users. This is the institutional knowledge layer — the collective brain of the plant.

Plant Memory is not personal. It belongs to the organization. Any authorized user can read it. Writing requires appropriate role permissions. All entries go through a verification workflow before becoming "official."

#### Tables

**`PlantLessonsLearned`** — Hard-won knowledge from actual field experience

| Column | Type | Description |
|--------|------|-------------|
| LessonID | INT IDENTITY | Primary key. |
| EquipmentClass | NVARCHAR(128) | Equipment class — Pump, Heat Exchanger, Compressor, Valve, etc. |
| EquipmentTag | NVARCHAR(64) | Specific equipment tag — P-4401A, HX-4401, K-2201. NULL if class-wide. |
| FailureMode | NVARCHAR(256) | What went wrong — "Seal failure", "Bearing overheating", "Tube leak", etc. |
| LessonText | NVARCHAR(4000) | The actual lesson in plain language. |
| SourceWO | NVARCHAR(64) | Work order number that prompted the lesson, if applicable. |
| SourceTurnaround | NVARCHAR(128) | Turnaround event name, if applicable. |
| AddedBy | NVARCHAR(128) | UserID of the person who added the lesson. |
| AddedByName | NVARCHAR(256) | Display name for readability. |
| AddedDate | DATETIME | When submitted. |
| VerifiedBy | NVARCHAR(128) | UserID of the lead planner/engineer who verified. NULL if not yet verified. |
| VerifiedDate | DATETIME | When verified. NULL if pending. |
| IsVerified | BIT | Whether the lesson has been reviewed and approved. |
| UseCount | INT | How many times this lesson has been surfaced in a bot response. |
| LastUsedDate | DATETIME | When last surfaced. |
| Tags | NVARCHAR(512) | Comma-separated tags for additional categorization. |
| IsActive | BIT | Soft delete flag. |

**`ApprovedProcedures`** — Standardized, versioned procedures for common tasks

| Column | Type | Description |
|--------|------|-------------|
| ProcedureID | INT IDENTITY | Primary key. |
| ProcedureTitle | NVARCHAR(256) | Descriptive title — "Centrifugal Pump Mechanical Seal Replacement", etc. |
| EquipmentClass | NVARCHAR(128) | Equipment class this applies to. |
| TaskType | NVARCHAR(128) | Type of task — PM, CM, Inspection, Turnaround, etc. |
| Steps | NVARCHAR(MAX) | JSON array of steps. Each step: {stepNumber, description, estimatedMinutes, safetyNotes, toolsRequired}. |
| Prerequisites | NVARCHAR(2000) | What must be true before starting. |
| SafetyWarnings | NVARCHAR(2000) | Critical safety information. |
| EstimatedDuration | DECIMAL(6,2) | Total estimated hours. |
| CreatedBy | NVARCHAR(128) | UserID. |
| ApprovedBy | NVARCHAR(128) | UserID of approver. |
| ApprovedDate | DATETIME | When approved. |
| Version | INT | Version number. Increments on each update. |
| PreviousVersionID | INT | FK to the previous version of this procedure, if any. |
| IsActive | BIT | Only the current active version has IsActive = 1. |

**`PlantGlossary`** — Plant-specific terminology and shorthand

| Column | Type | Description |
|--------|------|-------------|
| TermID | INT IDENTITY | Primary key. |
| Term | NVARCHAR(256) | The shorthand or informal term — "the big HX", "Charlie compressor", "south rack", etc. |
| Definition | NVARCHAR(1000) | What it actually means — "HX-4401 (Main Process Heat Exchanger)", etc. |
| Context | NVARCHAR(512) | Where/when this term is used — "Used by VN-PM planners", "Turnaround shorthand", etc. |
| EquipmentTag | NVARCHAR(64) | Linked equipment tag, if applicable. |
| AddedBy | NVARCHAR(128) | UserID. |
| AddedDate | DATETIME | When added. |
| IsActive | BIT | Soft delete. |

**`EquipmentNotes`** — Quick tips, warnings, and historical notes on specific equipment

| Column | Type | Description |
|--------|------|-------------|
| NoteID | INT IDENTITY | Primary key. |
| EquipmentTag | NVARCHAR(64) | The specific equipment tag this note applies to. |
| EquipmentClass | NVARCHAR(128) | Equipment class for broader queries. |
| NoteText | NVARCHAR(4000) | The note content. |
| NoteType | NVARCHAR(32) | Category: "tip", "warning", "history", "vendor", "specification". |
| Severity | NVARCHAR(16) | For warnings: "info", "caution", "critical". NULL for non-warnings. |
| AddedBy | NVARCHAR(128) | UserID. |
| AddedByName | NVARCHAR(256) | Display name. |
| AddedDate | DATETIME | When added. |
| VerifiedBy | NVARCHAR(128) | UserID of verifier. |
| VerifiedDate | DATETIME | When verified. |
| IsVerified | BIT | Verification flag. |
| UseCount | INT | Times surfaced. |
| IsActive | BIT | Soft delete. |

#### How Lessons Get Added

The capture flow is designed to be frictionless. It happens at the natural conclusion of work, not as a separate administrative task.

**Scenario:** A planner just finished writing a job plan for replacing the mechanical seal on pump P-4401A. During the process, they remembered that the last time this was done, the crew had to use an oversized seat because the bore had been machined during a previous repair.

1. Planner completes their task in LIAM
2. LIAM detects the conversation is winding down (or the planner explicitly says "I have a lesson to save")
3. LIAM prompts: "Did you learn anything during this task worth saving for future planners?"
4. Planner types: "P-4401A needs oversized mechanical seal seat — bore was machined in 2019. Standard seat will not seal properly."
5. LIAM confirms and writes to `PlantLessonsLearned`:
   - EquipmentTag: P-4401A
   - EquipmentClass: Pump
   - FailureMode: Seal failure
   - LessonText: "P-4401A needs oversized mechanical seal seat — bore was machined in 2019. Standard seat will not seal properly."
   - IsVerified: 0 (pending verification)
6. The entry is flagged for review by the lead planner

One sentence. Ten seconds of the planner's time. That knowledge is now permanent and searchable.

#### How Plant Memory Is Retrieved

When a user asks about specific equipment, failure modes, or procedures, the bot queries Plant Memory and injects relevant entries into the AI context alongside any other data (work orders, schedules, etc.).

**Retrieval logic:**

1. Parse the user query for equipment tags (regex pattern matching: `[A-Z]{1,4}-\d{3,5}[A-Z]?`)
2. Parse for equipment class keywords (pump, heat exchanger, compressor, valve, etc.)
3. Parse for failure mode keywords (seal, bearing, leak, vibration, corrosion, etc.)
4. Query `PlantLessonsLearned` WHERE (EquipmentTag matches OR (EquipmentClass matches AND FailureMode matches)) AND IsActive = 1, ORDER BY IsVerified DESC, UseCount DESC, AddedDate DESC, TOP 5
5. Query `EquipmentNotes` WHERE EquipmentTag matches AND IsActive = 1, ORDER BY NoteType (warnings first), AddedDate DESC, TOP 5
6. Query `PlantGlossary` for any terms that appear in the user query
7. Format results and inject into AI context
8. Increment UseCount on any entries that were surfaced

**Example injection:**

```
[PLANT MEMORY]
Equipment: P-4401A (Centrifugal Pump)

Lessons Learned (2 matches):
  1. [VERIFIED] "P-4401A needs oversized mechanical seal seat — bore was machined
     in 2019. Standard seat will not seal properly."
     — Added by M. Birklett, 2025-11-14 | Source WO: 200145823

  2. [VERIFIED] "When replacing seals on P-4401A, use John Crane Type 2100 only.
     The 1600 series does not fit the modified stuffing box."
     — Added by J. Hernandez, 2025-06-22 | Source WO: 200139847

Equipment Notes (1 match):
  1. [WARNING - CRITICAL] "P-4401A has modified bearing housing — do NOT order
     standard replacement housing. Contact Flowserve rep for modified drawing
     (DWG-P4401A-REV3)."
     — Added by T. Williams, 2024-12-03 | Verified by R. Chen, 2025-01-15
[/PLANT MEMORY]
```

#### Verification Workflow

Unverified entries still surface in results but are clearly marked as `[UNVERIFIED]`. This allows new knowledge to be immediately useful while maintaining quality control.

**Verification process:**

1. New lesson or note is added with `IsVerified = 0`
2. A Power Automate flow sends an Adaptive Card to the designated verifier (lead planner or reliability engineer) via Teams
3. The card shows the entry and offers: Approve, Reject, Edit & Approve
4. On approval: `IsVerified = 1`, `VerifiedBy` and `VerifiedDate` are set
5. On rejection: `IsActive = 0` (soft deleted), with a reason logged
6. On edit: the verifier can correct the text before approving

Unverified entries older than 30 days without action are flagged for attention. Entries rejected twice are permanently deactivated.

---

## 4. Full Database Schema

Complete SQL Server CREATE TABLE statements for all eight tables.

```sql
-- ============================================================
-- COPILOT MEMORY LAYER — FULL DATABASE SCHEMA
-- SQL Server 2019+ / Azure SQL Database
-- ============================================================

-- ============================================================
-- LAYER 1: USER MEMORY
-- ============================================================

-- ------------------------------------------------------------
-- Table: UserProfile
-- Core user identity and summary information.
-- Populated automatically from M365 authentication on first use.
-- ------------------------------------------------------------
CREATE TABLE dbo.UserProfile (
    UserID              NVARCHAR(128)   NOT NULL,
    DisplayName         NVARCHAR(256)   NOT NULL,
    Email               NVARCHAR(256)   NULL,
    Role                NVARCHAR(128)   NULL,
    Department          NVARCHAR(128)   NULL,
    Plant               NVARCHAR(128)   NULL,
    PreferredEquipmentClasses NVARCHAR(512) NULL,
    FrequentTopics      NVARCHAR(512)   NULL,
    OnboardedDate       DATETIME        NOT NULL DEFAULT GETDATE(),
    LastActiveDate      DATETIME        NOT NULL DEFAULT GETDATE(),
    SessionCount        INT             NOT NULL DEFAULT 0,
    IsActive            BIT             NOT NULL DEFAULT 1,
    CreatedDate         DATETIME        NOT NULL DEFAULT GETDATE(),
    ModifiedDate        DATETIME        NOT NULL DEFAULT GETDATE(),

    CONSTRAINT PK_UserProfile PRIMARY KEY (UserID)
);

CREATE INDEX IX_UserProfile_Email ON dbo.UserProfile (Email);
CREATE INDEX IX_UserProfile_Department ON dbo.UserProfile (Department);
CREATE INDEX IX_UserProfile_Plant ON dbo.UserProfile (Plant);
CREATE INDEX IX_UserProfile_IsActive ON dbo.UserProfile (IsActive);

-- ------------------------------------------------------------
-- Table: UserQueryHistory
-- Every question a user asks, logged silently in background.
-- Retention: default 2 years, configurable.
-- ------------------------------------------------------------
CREATE TABLE dbo.UserQueryHistory (
    QueryID             INT IDENTITY(1,1) NOT NULL,
    UserID              NVARCHAR(128)   NOT NULL,
    Timestamp           DATETIME        NOT NULL DEFAULT GETDATE(),
    QuestionText        NVARCHAR(2000)  NOT NULL,
    TopicCategory       NVARCHAR(128)   NULL,
    EquipmentReferenced NVARCHAR(256)   NULL,
    AgentName           NVARCHAR(128)   NULL,
    WasHelpful          BIT             NULL,
    ResponseSummary     NVARCHAR(1000)  NULL,

    CONSTRAINT PK_UserQueryHistory PRIMARY KEY (QueryID),
    CONSTRAINT FK_UserQueryHistory_UserProfile
        FOREIGN KEY (UserID) REFERENCES dbo.UserProfile(UserID)
);

CREATE INDEX IX_UserQueryHistory_UserID ON dbo.UserQueryHistory (UserID);
CREATE INDEX IX_UserQueryHistory_Timestamp ON dbo.UserQueryHistory (Timestamp DESC);
CREATE INDEX IX_UserQueryHistory_UserID_Timestamp
    ON dbo.UserQueryHistory (UserID, Timestamp DESC);
CREATE INDEX IX_UserQueryHistory_TopicCategory ON dbo.UserQueryHistory (TopicCategory);
CREATE INDEX IX_UserQueryHistory_EquipmentReferenced
    ON dbo.UserQueryHistory (EquipmentReferenced);
CREATE INDEX IX_UserQueryHistory_AgentName ON dbo.UserQueryHistory (AgentName);

-- ------------------------------------------------------------
-- Table: UserSavedQueries
-- User-bookmarked queries they want to reuse across sessions.
-- ------------------------------------------------------------
CREATE TABLE dbo.UserSavedQueries (
    SavedQueryID        INT IDENTITY(1,1) NOT NULL,
    UserID              NVARCHAR(128)   NOT NULL,
    QueryName           NVARCHAR(256)   NOT NULL,
    QueryText           NVARCHAR(2000)  NOT NULL,
    AgentName           NVARCHAR(128)   NULL,
    CreatedDate         DATETIME        NOT NULL DEFAULT GETDATE(),
    LastUsedDate        DATETIME        NULL,
    UseCount            INT             NOT NULL DEFAULT 0,
    IsActive            BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_UserSavedQueries PRIMARY KEY (SavedQueryID),
    CONSTRAINT FK_UserSavedQueries_UserProfile
        FOREIGN KEY (UserID) REFERENCES dbo.UserProfile(UserID)
);

CREATE INDEX IX_UserSavedQueries_UserID ON dbo.UserSavedQueries (UserID);
CREATE INDEX IX_UserSavedQueries_UserID_UseCount
    ON dbo.UserSavedQueries (UserID, UseCount DESC);
CREATE INDEX IX_UserSavedQueries_IsActive ON dbo.UserSavedQueries (IsActive);

-- ------------------------------------------------------------
-- Table: UserPreferences
-- Flexible key-value store for user-specific settings.
-- ------------------------------------------------------------
CREATE TABLE dbo.UserPreferences (
    PreferenceID        INT IDENTITY(1,1) NOT NULL,
    UserID              NVARCHAR(128)   NOT NULL,
    PreferenceName      NVARCHAR(256)   NOT NULL,
    PreferenceValue     NVARCHAR(1000)  NULL,
    UpdatedDate         DATETIME        NOT NULL DEFAULT GETDATE(),

    CONSTRAINT PK_UserPreferences PRIMARY KEY (PreferenceID),
    CONSTRAINT FK_UserPreferences_UserProfile
        FOREIGN KEY (UserID) REFERENCES dbo.UserProfile(UserID),
    CONSTRAINT UQ_UserPreferences_UserID_Name
        UNIQUE (UserID, PreferenceName)
);

CREATE INDEX IX_UserPreferences_UserID ON dbo.UserPreferences (UserID);

-- ============================================================
-- LAYER 2: PLANT MEMORY (INSTITUTIONAL KNOWLEDGE)
-- ============================================================

-- ------------------------------------------------------------
-- Table: PlantLessonsLearned
-- Hard-won knowledge from field experience.
-- Shared across all users. Write requires planner role.
-- Verification workflow before entries become "official."
-- ------------------------------------------------------------
CREATE TABLE dbo.PlantLessonsLearned (
    LessonID            INT IDENTITY(1,1) NOT NULL,
    EquipmentClass      NVARCHAR(128)   NULL,
    EquipmentTag        NVARCHAR(64)    NULL,
    FailureMode         NVARCHAR(256)   NULL,
    LessonText          NVARCHAR(4000)  NOT NULL,
    SourceWO            NVARCHAR(64)    NULL,
    SourceTurnaround    NVARCHAR(128)   NULL,
    AddedBy             NVARCHAR(128)   NOT NULL,
    AddedByName         NVARCHAR(256)   NULL,
    AddedDate           DATETIME        NOT NULL DEFAULT GETDATE(),
    VerifiedBy          NVARCHAR(128)   NULL,
    VerifiedDate        DATETIME        NULL,
    IsVerified          BIT             NOT NULL DEFAULT 0,
    UseCount            INT             NOT NULL DEFAULT 0,
    LastUsedDate        DATETIME        NULL,
    Tags                NVARCHAR(512)   NULL,
    IsActive            BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_PlantLessonsLearned PRIMARY KEY (LessonID)
);

CREATE INDEX IX_PlantLessonsLearned_EquipmentTag
    ON dbo.PlantLessonsLearned (EquipmentTag);
CREATE INDEX IX_PlantLessonsLearned_EquipmentClass
    ON dbo.PlantLessonsLearned (EquipmentClass);
CREATE INDEX IX_PlantLessonsLearned_FailureMode
    ON dbo.PlantLessonsLearned (FailureMode);
CREATE INDEX IX_PlantLessonsLearned_IsVerified
    ON dbo.PlantLessonsLearned (IsVerified);
CREATE INDEX IX_PlantLessonsLearned_IsActive
    ON dbo.PlantLessonsLearned (IsActive);
CREATE INDEX IX_PlantLessonsLearned_EquipmentClass_FailureMode
    ON dbo.PlantLessonsLearned (EquipmentClass, FailureMode);
CREATE INDEX IX_PlantLessonsLearned_AddedBy
    ON dbo.PlantLessonsLearned (AddedBy);

-- ------------------------------------------------------------
-- Table: ApprovedProcedures
-- Standardized, versioned procedures for common maintenance tasks.
-- Steps stored as JSON for flexibility.
-- ------------------------------------------------------------
CREATE TABLE dbo.ApprovedProcedures (
    ProcedureID         INT IDENTITY(1,1) NOT NULL,
    ProcedureTitle      NVARCHAR(256)   NOT NULL,
    EquipmentClass      NVARCHAR(128)   NOT NULL,
    TaskType            NVARCHAR(128)   NOT NULL,
    Steps               NVARCHAR(MAX)   NOT NULL,  -- JSON array
    Prerequisites       NVARCHAR(2000)  NULL,
    SafetyWarnings      NVARCHAR(2000)  NULL,
    EstimatedDuration   DECIMAL(6,2)    NULL,
    CreatedBy           NVARCHAR(128)   NOT NULL,
    CreatedDate         DATETIME        NOT NULL DEFAULT GETDATE(),
    ApprovedBy          NVARCHAR(128)   NULL,
    ApprovedDate        DATETIME        NULL,
    Version             INT             NOT NULL DEFAULT 1,
    PreviousVersionID   INT             NULL,
    IsActive            BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_ApprovedProcedures PRIMARY KEY (ProcedureID),
    CONSTRAINT FK_ApprovedProcedures_PreviousVersion
        FOREIGN KEY (PreviousVersionID) REFERENCES dbo.ApprovedProcedures(ProcedureID)
);

CREATE INDEX IX_ApprovedProcedures_EquipmentClass
    ON dbo.ApprovedProcedures (EquipmentClass);
CREATE INDEX IX_ApprovedProcedures_TaskType
    ON dbo.ApprovedProcedures (TaskType);
CREATE INDEX IX_ApprovedProcedures_EquipmentClass_TaskType
    ON dbo.ApprovedProcedures (EquipmentClass, TaskType);
CREATE INDEX IX_ApprovedProcedures_IsActive
    ON dbo.ApprovedProcedures (IsActive);
CREATE INDEX IX_ApprovedProcedures_Version
    ON dbo.ApprovedProcedures (Version);

-- ------------------------------------------------------------
-- Table: PlantGlossary
-- Plant-specific terminology, nicknames, and shorthand.
-- Maps informal language to formal equipment tags and terms.
-- ------------------------------------------------------------
CREATE TABLE dbo.PlantGlossary (
    TermID              INT IDENTITY(1,1) NOT NULL,
    Term                NVARCHAR(256)   NOT NULL,
    Definition          NVARCHAR(1000)  NOT NULL,
    Context             NVARCHAR(512)   NULL,
    EquipmentTag        NVARCHAR(64)    NULL,
    AddedBy             NVARCHAR(128)   NOT NULL,
    AddedDate           DATETIME        NOT NULL DEFAULT GETDATE(),
    IsActive            BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_PlantGlossary PRIMARY KEY (TermID)
);

CREATE INDEX IX_PlantGlossary_Term ON dbo.PlantGlossary (Term);
CREATE INDEX IX_PlantGlossary_EquipmentTag ON dbo.PlantGlossary (EquipmentTag);
CREATE INDEX IX_PlantGlossary_IsActive ON dbo.PlantGlossary (IsActive);

-- Full-text index for natural language search on Term and Definition
-- (Requires full-text catalog — create if not exists)
-- CREATE FULLTEXT CATALOG FTC_PlantMemory AS DEFAULT;
-- CREATE FULLTEXT INDEX ON dbo.PlantGlossary (Term, Definition)
--     KEY INDEX PK_PlantGlossary ON FTC_PlantMemory;

-- ------------------------------------------------------------
-- Table: EquipmentNotes
-- Quick tips, warnings, and historical context on specific equipment.
-- The "sticky notes" of the plant — digitized and searchable.
-- ------------------------------------------------------------
CREATE TABLE dbo.EquipmentNotes (
    NoteID              INT IDENTITY(1,1) NOT NULL,
    EquipmentTag        NVARCHAR(64)    NOT NULL,
    EquipmentClass      NVARCHAR(128)   NULL,
    NoteText            NVARCHAR(4000)  NOT NULL,
    NoteType            NVARCHAR(32)    NOT NULL,  -- tip, warning, history, vendor, specification
    Severity            NVARCHAR(16)    NULL,       -- info, caution, critical (for warnings)
    AddedBy             NVARCHAR(128)   NOT NULL,
    AddedByName         NVARCHAR(256)   NULL,
    AddedDate           DATETIME        NOT NULL DEFAULT GETDATE(),
    VerifiedBy          NVARCHAR(128)   NULL,
    VerifiedDate        DATETIME        NULL,
    IsVerified          BIT             NOT NULL DEFAULT 0,
    UseCount            INT             NOT NULL DEFAULT 0,
    IsActive            BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_EquipmentNotes PRIMARY KEY (NoteID),
    CONSTRAINT CK_EquipmentNotes_NoteType
        CHECK (NoteType IN ('tip', 'warning', 'history', 'vendor', 'specification')),
    CONSTRAINT CK_EquipmentNotes_Severity
        CHECK (Severity IS NULL OR Severity IN ('info', 'caution', 'critical'))
);

CREATE INDEX IX_EquipmentNotes_EquipmentTag ON dbo.EquipmentNotes (EquipmentTag);
CREATE INDEX IX_EquipmentNotes_EquipmentClass ON dbo.EquipmentNotes (EquipmentClass);
CREATE INDEX IX_EquipmentNotes_NoteType ON dbo.EquipmentNotes (NoteType);
CREATE INDEX IX_EquipmentNotes_IsVerified ON dbo.EquipmentNotes (IsVerified);
CREATE INDEX IX_EquipmentNotes_IsActive ON dbo.EquipmentNotes (IsActive);
CREATE INDEX IX_EquipmentNotes_EquipmentTag_NoteType
    ON dbo.EquipmentNotes (EquipmentTag, NoteType);

-- ============================================================
-- OPTIONAL: Full-Text Search Catalog
-- Enables natural language search across lessons and notes.
-- Uncomment and run separately if full-text search is available.
-- ============================================================

-- CREATE FULLTEXT CATALOG FTC_PlantMemory AS DEFAULT;
--
-- CREATE FULLTEXT INDEX ON dbo.PlantLessonsLearned
--     (LessonText, FailureMode, Tags)
--     KEY INDEX PK_PlantLessonsLearned ON FTC_PlantMemory;
--
-- CREATE FULLTEXT INDEX ON dbo.EquipmentNotes
--     (NoteText)
--     KEY INDEX PK_EquipmentNotes ON FTC_PlantMemory;
--
-- CREATE FULLTEXT INDEX ON dbo.PlantGlossary
--     (Term, Definition)
--     KEY INDEX PK_PlantGlossary ON FTC_PlantMemory;

-- ============================================================
-- UTILITY VIEWS
-- ============================================================

-- View: Active lessons with verification status
CREATE VIEW dbo.vw_ActiveLessons AS
SELECT
    l.LessonID,
    l.EquipmentClass,
    l.EquipmentTag,
    l.FailureMode,
    l.LessonText,
    l.SourceWO,
    l.AddedByName,
    l.AddedDate,
    l.IsVerified,
    CASE WHEN l.IsVerified = 1 THEN 'VERIFIED' ELSE 'UNVERIFIED' END AS VerificationStatus,
    l.UseCount,
    l.Tags
FROM dbo.PlantLessonsLearned l
WHERE l.IsActive = 1;
GO

-- View: Active equipment notes, warnings first
CREATE VIEW dbo.vw_EquipmentNotesRanked AS
SELECT
    n.NoteID,
    n.EquipmentTag,
    n.EquipmentClass,
    n.NoteText,
    n.NoteType,
    n.Severity,
    n.AddedByName,
    n.AddedDate,
    n.IsVerified,
    CASE
        WHEN n.NoteType = 'warning' AND n.Severity = 'critical' THEN 1
        WHEN n.NoteType = 'warning' AND n.Severity = 'caution' THEN 2
        WHEN n.NoteType = 'warning' AND n.Severity = 'info' THEN 3
        WHEN n.NoteType = 'tip' THEN 4
        WHEN n.NoteType = 'specification' THEN 5
        WHEN n.NoteType = 'vendor' THEN 6
        WHEN n.NoteType = 'history' THEN 7
        ELSE 8
    END AS SortPriority
FROM dbo.EquipmentNotes n
WHERE n.IsActive = 1;
GO

-- View: User session summary for context loading
CREATE VIEW dbo.vw_UserSessionContext AS
SELECT
    u.UserID,
    u.DisplayName,
    u.Role,
    u.Department,
    u.Plant,
    u.PreferredEquipmentClasses,
    u.FrequentTopics,
    u.SessionCount,
    u.LastActiveDate,
    u.OnboardedDate
FROM dbo.UserProfile u
WHERE u.IsActive = 1;
GO
```

---

## 5. Power Automate Flows

Five Power Automate flows support the memory layer. Each is designed to be reusable across any Copilot Studio agent.

### Flow 1: Session Start — Load User Context

**Purpose:** Load the full user context at the beginning of every conversation and return it as a formatted string for system prompt injection.

**Trigger:** Called as a Copilot Studio action (`GetUserContext`) at conversation start.

**Input:** `UserID` (from M365 authentication token via `System.User.Id`)

**Steps:**

1. **Initialize variable** `contextString` (String, empty)
2. **SQL Connector — Execute Query:** Check if user exists in `UserProfile`
   ```sql
   SELECT * FROM dbo.UserProfile WHERE UserID = @UserID AND IsActive = 1
   ```
3. **Condition:** If user exists → continue. If not → **SQL Connector — Insert Row** to create new UserProfile from M365 Graph data, then continue.
4. **SQL Connector — Execute Query:** Update session metadata
   ```sql
   UPDATE dbo.UserProfile
   SET LastActiveDate = GETDATE(), SessionCount = SessionCount + 1, ModifiedDate = GETDATE()
   WHERE UserID = @UserID
   ```
5. **SQL Connector — Execute Query:** Get recent queries (last 7 days, top 5)
   ```sql
   SELECT TOP 5 QuestionText, Timestamp, TopicCategory, EquipmentReferenced
   FROM dbo.UserQueryHistory
   WHERE UserID = @UserID AND Timestamp >= DATEADD(day, -7, GETDATE())
   ORDER BY Timestamp DESC
   ```
6. **SQL Connector — Execute Query:** Get saved queries (top 5 by UseCount)
   ```sql
   SELECT TOP 5 QueryName, QueryText, UseCount
   FROM dbo.UserSavedQueries
   WHERE UserID = @UserID AND IsActive = 1
   ORDER BY UseCount DESC
   ```
7. **SQL Connector — Execute Query:** Get user preferences
   ```sql
   SELECT PreferenceName, PreferenceValue
   FROM dbo.UserPreferences
   WHERE UserID = @UserID
   ```
8. **Compose:** Build the formatted context string from all query results
9. **Return** `contextString` to Copilot Studio

**Output:** Formatted context string (see Section 6 for exact format).

**Performance target:** < 2 seconds total execution time. All SQL queries hit indexed columns.

---

### Flow 2: Query Logged

**Purpose:** Silently log every user query for history, analytics, and context building.

**Trigger:** Called as a Copilot Studio action (`LogQuery`) after each bot response.

**Input:** `UserID`, `QuestionText`, `TopicCategory` (auto-classified by Copilot), `EquipmentReferenced` (extracted by Copilot), `AgentName`, `ResponseSummary`

**Steps:**

1. **SQL Connector — Insert Row** into `UserQueryHistory`:
   ```sql
   INSERT INTO dbo.UserQueryHistory
       (UserID, QuestionText, TopicCategory, EquipmentReferenced, AgentName, ResponseSummary)
   VALUES
       (@UserID, @QuestionText, @TopicCategory, @EquipmentReferenced, @AgentName, @ResponseSummary)
   ```
2. **No return value.** This is a fire-and-forget background write.

**Performance target:** < 500ms. Single INSERT, no reads.

**Topic classification:** The Copilot agent uses its AI model to classify the query into one of the predefined categories before calling this flow: `Job Plan`, `Backlog`, `Scheduling`, `Equipment Lookup`, `Procedure`, `Notification`, `Turnaround`, `Reporting`, `General`. This classification is imperfect but useful for analytics and context building.

**Equipment extraction:** The Copilot agent uses regex and AI to extract any equipment tags mentioned in the query. Pattern: `[A-Z]{1,4}-\d{3,5}[A-Z]?` plus natural language references resolved via PlantGlossary.

---

### Flow 3: Plant Memory Retrieval

**Purpose:** Search Plant Memory for relevant lessons, notes, and glossary entries based on the current user query.

**Trigger:** Called as a Copilot Studio action (`GetPlantMemory`) when Copilot detects an equipment-specific or failure-mode-specific query.

**Input:** `EquipmentTag` (nullable), `EquipmentClass` (nullable), `FailureMode` (nullable), `SearchTerms` (nullable, free text)

**Steps:**

1. **SQL Connector — Execute Query:** Search PlantLessonsLearned
   ```sql
   SELECT TOP 5
       LessonID, EquipmentClass, EquipmentTag, FailureMode, LessonText,
       SourceWO, AddedByName, AddedDate, IsVerified, UseCount
   FROM dbo.PlantLessonsLearned
   WHERE IsActive = 1
     AND (
       (@EquipmentTag IS NOT NULL AND EquipmentTag = @EquipmentTag)
       OR (@EquipmentClass IS NOT NULL AND EquipmentClass = @EquipmentClass
           AND (@FailureMode IS NULL OR FailureMode LIKE '%' + @FailureMode + '%'))
     )
   ORDER BY
       CASE WHEN EquipmentTag = @EquipmentTag THEN 0 ELSE 1 END,
       IsVerified DESC,
       UseCount DESC,
       AddedDate DESC
   ```
2. **SQL Connector — Execute Query:** Search EquipmentNotes
   ```sql
   SELECT TOP 5
       NoteID, EquipmentTag, NoteText, NoteType, Severity,
       AddedByName, AddedDate, IsVerified
   FROM dbo.EquipmentNotes
   WHERE IsActive = 1
     AND (
       (@EquipmentTag IS NOT NULL AND EquipmentTag = @EquipmentTag)
       OR (@EquipmentClass IS NOT NULL AND EquipmentClass = @EquipmentClass)
     )
   ORDER BY
       CASE WHEN NoteType = 'warning' AND Severity = 'critical' THEN 1
            WHEN NoteType = 'warning' AND Severity = 'caution' THEN 2
            WHEN NoteType = 'warning' THEN 3
            ELSE 4 END,
       AddedDate DESC
   ```
3. **SQL Connector — Execute Query:** Search PlantGlossary (if SearchTerms provided)
   ```sql
   SELECT TermID, Term, Definition, Context
   FROM dbo.PlantGlossary
   WHERE IsActive = 1 AND Term LIKE '%' + @SearchTerms + '%'
   ```
4. **SQL Connector — Execute Query:** Update UseCount on returned lessons and notes
   ```sql
   UPDATE dbo.PlantLessonsLearned
   SET UseCount = UseCount + 1, LastUsedDate = GETDATE()
   WHERE LessonID IN (@ReturnedLessonIDs)
   ```
5. **Compose:** Format results into `[PLANT MEMORY]` block
6. **Return** formatted plant memory string to Copilot Studio

**Output:** Formatted plant memory context string (see Section 3 for format example).

**Performance target:** < 3 seconds. Multiple queries but all on indexed columns.

---

### Flow 4: Lesson Learned Capture

**Purpose:** Capture a lesson learned from a planner and store it for verification.

**Trigger:** Called as a Copilot Studio action (`SaveLesson`) when the user explicitly offers a lesson or responds to the end-of-session prompt.

**Input:** `UserID`, `UserDisplayName`, `EquipmentTag` (nullable), `EquipmentClass` (nullable), `FailureMode` (nullable), `LessonText`, `SourceWO` (nullable), `Tags` (nullable)

**Steps:**

1. **SQL Connector — Insert Row** into `PlantLessonsLearned`:
   ```sql
   INSERT INTO dbo.PlantLessonsLearned
       (EquipmentClass, EquipmentTag, FailureMode, LessonText, SourceWO,
        AddedBy, AddedByName, IsVerified, IsActive)
   VALUES
       (@EquipmentClass, @EquipmentTag, @FailureMode, @LessonText, @SourceWO,
        @UserID, @UserDisplayName, 0, 1)
   ```
2. **Get the inserted LessonID** (using SCOPE_IDENTITY or separate query)
3. **Send Adaptive Card** via Teams to the designated verifier:
   - Card shows: Equipment tag, failure mode, lesson text, source WO, who added it
   - Buttons: Approve, Reject, Edit & Approve
   - Card includes LessonID for callback
4. **Return** confirmation message to Copilot Studio: "Lesson saved and sent to [Verifier Name] for review."

**Verifier assignment logic:** The flow looks up the designated verifier for the equipment class or department. This is configurable — default is the lead planner for the department of the user who submitted the lesson.

---

### Flow 5: Saved Query

**Purpose:** Save a query that the user wants to reuse.

**Trigger:** Called as a Copilot Studio action (`SaveQuery`) when the user says "save this query", "bookmark this", or similar.

**Input:** `UserID`, `QueryName` (user-provided or auto-generated), `QueryText`, `AgentName`

**Steps:**

1. **Condition:** Check if user already has a saved query with the same name
   ```sql
   SELECT SavedQueryID FROM dbo.UserSavedQueries
   WHERE UserID = @UserID AND QueryName = @QueryName AND IsActive = 1
   ```
2. **If exists:** Update the existing entry (new QueryText, reset UseCount if text changed)
3. **If not:** Insert new row
   ```sql
   INSERT INTO dbo.UserSavedQueries
       (UserID, QueryName, QueryText, AgentName, UseCount)
   VALUES
       (@UserID, @QueryName, @QueryText, @AgentName, 0)
   ```
4. **Return** confirmation: "Query saved as '[QueryName]'. Say 'run [QueryName]' anytime to execute it."

**Auto-naming logic:** If the user does not provide a name, the Copilot agent generates one based on the query content (e.g., "pump seal job plan" becomes "Pump Seal Job Plan"). The user can override.

---

## 6. Copilot Studio Integration

This section describes how to wire the memory layer into any Copilot Studio agent. The integration points are: system prompt injection, action definitions, and session start topic flow.

### System Prompt Injection

The user context string is injected into the system prompt at session start. This gives the AI model full awareness of who the user is and what they have been working on.

**Format:**

```
[USER CONTEXT]
Name: {DisplayName}
Role: {Role} | Department: {Department} | Plant: {Plant}
Top equipment classes: {PreferredEquipmentClasses}
Session count: {SessionCount} | Last active: {LastActiveDate}

Recent queries (last 7 days):
  1. "{QuestionText}" ({Timestamp})
  2. "{QuestionText}" ({Timestamp})
  3. "{QuestionText}" ({Timestamp})
  4. "{QuestionText}" ({Timestamp})
  5. "{QuestionText}" ({Timestamp})

Saved queries (by use count):
  1. "{QueryName}" (used {UseCount} times)
  2. "{QueryName}" (used {UseCount} times)
  3. "{QueryName}" (used {UseCount} times)
  4. "{QueryName}" (used {UseCount} times)
  5. "{QueryName}" (used {UseCount} times)

Preferences:
  {PreferenceName}: {PreferenceValue}
  {PreferenceName}: {PreferenceValue}

Plant memory matches for this session: [loaded per query via GetPlantMemory action]
[/USER CONTEXT]
```

**Injection method:** In the Copilot Studio system prompt (under "Instructions"), include a placeholder like `{{UserContext}}`. The session start topic flow sets the global variable `Global.UserContext` via the `GetUserContext` action, and the system prompt references it. The exact mechanism depends on the Copilot Studio version — in current versions, you include the variable in the Instructions field or pass it as a system message at the beginning of the conversation.

**Token budget:** The user context string is designed to stay under 1,000 tokens. This leaves the vast majority of the context window available for the actual conversation. Plant memory results (loaded per-query) add another 500-1,000 tokens when relevant.

### Action Definitions

Five Copilot Studio actions must be created. Each maps to one Power Automate flow.

| Action Name | Trigger | Input Parameters | Output |
|-------------|---------|------------------|--------|
| `GetUserContext` | Session start (topic flow) | UserID (from System.User.Id) | Formatted context string |
| `LogQuery` | After every bot response (topic flow) | UserID, QuestionText, TopicCategory, EquipmentReferenced, AgentName, ResponseSummary | None (fire-and-forget) |
| `GetPlantMemory` | When query references equipment or failure modes | EquipmentTag, EquipmentClass, FailureMode, SearchTerms | Formatted plant memory string |
| `SaveLesson` | User explicitly offers a lesson or end-of-session prompt | UserID, UserDisplayName, EquipmentTag, EquipmentClass, FailureMode, LessonText, SourceWO, Tags | Confirmation message |
| `SaveQuery` | User says "save this query" or similar | UserID, QueryName, QueryText, AgentName | Confirmation message |

**Creating the actions in Copilot Studio:**

1. Go to the agent's **Actions** panel
2. Click **Add an action** > **Power Automate flow**
3. Select the corresponding flow
4. Map input parameters from Copilot Studio variables
5. Map output to Copilot Studio variables (for actions that return data)

### Session Start Topic Flow

The session start topic is the entry point for every conversation. It must load user context before the bot sends its first message.

**Topic: Greeting (modified)**

```
Trigger: Conversation Start

Node 1: Action — Call GetUserContext
  Input: System.User.Id → UserID
  Output: contextString → Global.UserContext

Node 2: Condition — Is this a new user?
  If Global.UserContext contains "Session count: 1"
    → Node 3a: Message — "Welcome to [AgentName]. I'm setting up your profile now.
       What's your role? (e.g., Maintenance Planner, Scheduler, Reliability Engineer)"
    → Node 4a: Question — capture role, update UserProfile via additional flow
  Else
    → Node 3b: Message — personalized greeting using context
       "Welcome back, {DisplayName}. {ProactiveInsight}"

Node 5: Set system message with Global.UserContext injected

Node 6: Redirect to main conversation topic
```

**Proactive insight logic:** The greeting message can include a proactive insight based on the user's context. Examples:

- If the user has a saved query with high use count: "Your '{QueryName}' is ready to run."
- If the user's last session was about specific equipment: "Last time you were looking at {EquipmentReferenced}."
- If the user has been inactive for > 7 days: "It's been a while — here's a quick catch-up on your equipment."

These proactive elements are generated by the AI model based on the injected context, not hard-coded.

### Query Logging Integration

After every bot response, the `LogQuery` action fires in the background. This is implemented as a secondary topic flow or an "after response" hook.

**Implementation approach:**

In Copilot Studio, you cannot directly hook into "after every response." Instead, implement a wrapper topic that:

1. Receives the user message
2. Processes it through the normal AI orchestration
3. After the response is generated, calls `LogQuery` as a background action
4. The action call does not block the conversation — it fires and the user sees no delay

**Alternative:** Use a Power Automate flow triggered by Dataverse conversation logs (if using Dataverse) or a custom event mechanism. The exact approach depends on the Copilot Studio version and channel.

### Plant Memory Integration

Plant memory retrieval is triggered when the AI model detects that the user's query involves specific equipment or failure modes. This is implemented via the `GetPlantMemory` action, which the AI model can call as a tool/action during its reasoning process.

**In the system prompt:**

```
You have access to a Plant Memory database containing lessons learned, equipment notes,
and plant-specific terminology. When a user asks about specific equipment (identified by
tag like P-4401A or HX-4401) or about specific failure modes (seal failure, bearing issues,
tube leaks, etc.), call the GetPlantMemory action to retrieve relevant institutional
knowledge. Include any retrieved plant memory in your response, especially warnings and
critical notes.
```

This allows the AI model to decide when to query plant memory, rather than querying on every single message. The model is smart enough to skip plant memory queries for general questions like "what's on my backlog?" and invoke it for specific questions like "what do I need to know about replacing the seal on P-4401A?"

---

## 7. Dataverse Option

For organizations that prefer Dataverse over SQL Server — or that want tighter integration with the Power Platform ecosystem — the memory layer can be implemented in Dataverse with minimal changes.

### Table Equivalents in Dataverse

Each SQL table maps to a Dataverse table (entity). Naming convention follows Dataverse standards with a publisher prefix (e.g., `btk_`).

| SQL Table | Dataverse Table | Notes |
|-----------|----------------|-------|
| `UserProfile` | `btk_userprofile` | UserID becomes an alternate key; Dataverse auto-generates a GUID primary key |
| `UserQueryHistory` | `btk_userqueryhistory` | Lookup relationship to btk_userprofile |
| `UserSavedQueries` | `btk_usersavedqueries` | Lookup relationship to btk_userprofile |
| `UserPreferences` | `btk_userpreferences` | Lookup relationship to btk_userprofile |
| `PlantLessonsLearned` | `btk_plantlessonslearned` | Choice columns for EquipmentClass, NoteType |
| `ApprovedProcedures` | `btk_approvedprocedures` | Steps stored in multi-line text (JSON) |
| `PlantGlossary` | `btk_plantglossary` | |
| `EquipmentNotes` | `btk_equipmentnotes` | Choice columns for NoteType, Severity |

### Differences in Power Automate Connector

The Dataverse connector differs from the SQL Server connector in several ways:

- **No raw SQL:** You use "List rows" and "Get a row" actions with OData filter expressions instead of SQL queries
- **Filter syntax:** Instead of `WHERE UserID = @UserID`, you use `btk_userid eq '{UserID}'`
- **Relationships:** Dataverse natively supports lookup columns, so foreign key relationships are handled by the platform
- **Sorting:** OData `$orderby` instead of SQL `ORDER BY`
- **Pagination:** Dataverse returns max 5,000 rows per request and supports `$top` for limiting results

**Example — Load recent queries in Dataverse:**

```
Action: List rows
Table: btk_userqueryhistory
Filter: btk_userid eq '{UserID}' and btk_timestamp ge '{7DaysAgo}'
Sort: btk_timestamp desc
Row count: 5
```

### When to Choose Dataverse vs. SQL Server

| Criterion | SQL Server | Dataverse |
|-----------|-----------|-----------|
| **Query complexity** | Full T-SQL — JOINs, CTEs, window functions, full-text search | OData filters only — no JOINs, limited aggregation |
| **Performance** | Faster for complex queries, direct index control | Adequate for simple CRUD, throttling on high volume |
| **Cost** | Azure SQL Database pricing (can be cheaper for small workloads) | Included with Power Platform licenses (may already be paid for) |
| **Administration** | DBA required for schema changes, backups, tuning | Managed by Microsoft — no DBA needed |
| **Integration** | SQL connector in Power Automate (standard) | Native Dataverse connector (premium, but often already licensed) |
| **Security model** | Row-level security via SQL, managed separately | Dataverse security roles, integrated with M365 |
| **Offline/mobile** | No native offline | Dataverse supports offline-capable model-driven apps |
| **Auditability** | Custom audit logging required | Built-in audit trail |

**Recommendation:**

- Use **SQL Server** if you need complex queries, full-text search, or have a DBA on staff
- Use **Dataverse** if you are already deep in the Power Platform ecosystem, want zero database administration, and your queries are straightforward
- Start with **SQL Server** for the prototype (easier to iterate on schema) and migrate to Dataverse later if needed

### Migration Path

SQL Server to Dataverse migration:

1. Create Dataverse tables matching the SQL schema (use the Dataverse table designer or solution import)
2. Use a Power Automate flow or Azure Data Factory to migrate existing data
3. Update all Power Automate flows to use Dataverse connector instead of SQL connector
4. Update Copilot Studio actions to point to the new flows
5. Run both systems in parallel for 2 weeks to validate
6. Decommission SQL tables

The memory layer is designed so that the Copilot Studio agent does not know or care whether the backend is SQL or Dataverse. The agent interacts with actions that return formatted strings. The storage layer is abstracted behind the Power Automate flows.

---

## 8. Privacy & Access Control

### User Data — Personal and Protected

User Memory (Layer 1) data is personal to each user:

- **UserProfile:** Readable by the user and system admins. Not visible to other users.
- **UserQueryHistory:** Readable only by the user and system admins. This contains the full text of every question asked — treat it as sensitive.
- **UserSavedQueries:** Readable only by the user and system admins.
- **UserPreferences:** Readable only by the user and system admins.

**Enforcement mechanism:**

- In SQL Server: All queries filter on `UserID`, which is populated from the authenticated M365 token. The Power Automate flows enforce this — there is no direct user access to the database.
- In Dataverse: Row-level security rules based on the `Owner` field and Dataverse security roles.

No user can see another user's query history, saved queries, or preferences. Period.

### Plant Memory — Shared, Controlled Writes

Plant Memory (Layer 2) data is organizational knowledge:

- **Read access:** All authenticated users can read plant memory (lessons, notes, glossary, procedures). This is the point — the knowledge is shared.
- **Write access:** Only users with the "Planner" role (or higher) can add lessons, notes, and glossary entries. This is enforced in the Power Automate flows by checking the user's role in UserProfile before allowing the insert.
- **Verification:** New entries are unverified by default. Only designated verifiers (lead planners, reliability engineers) can approve entries. Verification is a separate action — the verifier receives an Adaptive Card in Teams and approves/rejects.
- **Approved Procedures:** Only users with the "Lead Planner" or "Reliability Engineer" role can create or modify approved procedures. Procedures require explicit approval before becoming active.

### Admin View

A Power Apps model-driven app or Canvas app provides an admin dashboard:

- View all users and their activity statistics
- Browse and manage plant memory entries (approve, reject, edit, deactivate)
- View verification queue (pending entries)
- View analytics: most queried equipment, most used saved queries, lesson capture rate
- Manage verifier assignments
- Configure retention policies

Admin access is restricted to users with the "Admin" or "System Administrator" role in the memory layer.

### Data Retention

Configurable retention policies:

| Data Type | Default Retention | Rationale |
|-----------|-------------------|-----------|
| UserProfile | Indefinite (soft delete on deactivation) | Core identity, low volume |
| UserQueryHistory | 2 years | High volume, analytics value diminishes over time |
| UserSavedQueries | Indefinite (user-managed) | Low volume, user owns the lifecycle |
| UserPreferences | Indefinite | Low volume |
| PlantLessonsLearned | Indefinite | Institutional knowledge — never delete |
| ApprovedProcedures | Indefinite (versioned) | Old versions deactivated but retained |
| PlantGlossary | Indefinite | Low volume |
| EquipmentNotes | Indefinite | Institutional knowledge — never delete |

**Automated cleanup:** A scheduled Power Automate flow runs monthly to:

1. Delete `UserQueryHistory` records older than the retention period
2. Flag inactive users (no session in 180 days) for review
3. Flag unverified lessons/notes older than 30 days for verifier attention
4. Generate a retention report for admins

### GDPR and Compliance Considerations

- **Right to access:** Users can request a full export of their personal data (UserProfile, UserQueryHistory, UserSavedQueries, UserPreferences) via the admin dashboard or an automated flow
- **Right to erasure:** Users can request deletion of their personal data. This removes all Layer 1 data. Layer 2 contributions (lessons, notes) are anonymized but not deleted, as they are organizational knowledge
- **Data location:** SQL Server or Dataverse instance should be in the same geographic region as the organization's M365 tenant
- **Encryption:** Data at rest is encrypted by Azure SQL Database or Dataverse by default. Data in transit is encrypted via TLS
- **Audit trail:** All writes to plant memory are attributed (AddedBy, VerifiedBy with timestamps). For SQL Server, enable SQL Audit for compliance-grade logging. For Dataverse, built-in auditing covers this
- **Consent:** Users are informed on first interaction that query history is logged for personalization. They can opt out of query logging via a preference setting (`QueryLoggingOptOut: true`), which causes the LogQuery flow to skip the insert

---

## 9. LIAM + P6 Intelligence Integration Examples

The memory layer is agent-agnostic, but its value is best illustrated through concrete examples with BAITEKS agents.

### LIAM — User Memory Example

**Scenario:** Mike Birklett opens LIAM on a Tuesday morning.

The `GetUserContext` flow fires and returns:

```
[USER CONTEXT]
Name: Mike Birklett
Role: Maintenance Planner | Department: VN-PM | Plant: Valero Norco
Top equipment classes: Pumps, Heat Exchangers, Compressors
Session count: 148 | Last active: 2026-03-05

Recent queries (last 7 days):
  1. "what's on the VN-PM backlog for this week" (2026-03-05)
  2. "pump mechanical seal job plan P-4401A" (2026-03-05)
  3. "HX-4401 bundle removal steps" (2026-03-04)
  4. "compressor K-2201 bearing failure history" (2026-03-03)
  5. "open pump notifications VN-PM" (2026-03-03)

Saved queries (by use count):
  1. "VN-PM Daily Summary" (used 89 times)
  2. "Pump Seal Job Plan Template" (used 34 times)
  3. "Open HX Work Orders" (used 28 times)
[/USER CONTEXT]
```

LIAM responds:

> "Welcome back, Mike. Your VN-PM Daily Summary is ready — want me to run it? Also, since yesterday, there are 2 new pump notifications and the HX-4401 job plan you were working on has been approved by the lead planner."

Mike says: "Run the daily summary."

LIAM executes the saved query, retrieves the backlog from SAP PM, and presents a structured summary. The `LogQuery` flow logs this interaction. The `UseCount` on "VN-PM Daily Summary" increments to 90.

**Why this matters:** Mike did not have to re-explain who he is, what department he is in, or what he cares about. The bot picked up exactly where it left off. It even proactively mentioned the notifications it knew he would care about (based on his equipment class preferences).

### LIAM — Plant Memory Example

**Scenario:** A relatively new planner, Sarah, asks LIAM about replacing the seal on pump P-4401A.

Sarah types: "I need to create a job plan for mechanical seal replacement on P-4401A. What do I need to know?"

LIAM calls `GetPlantMemory` with EquipmentTag = "P-4401A", EquipmentClass = "Pump", FailureMode = "Seal".

The flow returns:

```
[PLANT MEMORY]
Equipment: P-4401A (Centrifugal Pump)

Lessons Learned (3 matches):
  1. [VERIFIED] "P-4401A needs oversized mechanical seal seat — bore was machined
     in 2019 during last major repair. Standard seat will not seal properly. Confirm
     seat OD before ordering."
     — M. Birklett, 2025-11-14 | WO: 200145823

  2. [VERIFIED] "When replacing seals on P-4401A, use John Crane Type 2100 only.
     The 1600 series does not fit the modified stuffing box."
     — J. Hernandez, 2025-06-22 | WO: 200139847

  3. [VERIFIED] "P-4401A coupling alignment is critical after seal work — use laser
     alignment, not dial indicators. Soft foot on outboard bearing housing."
     — T. Williams, 2025-03-10 | WO: 200132156

Equipment Notes (2 matches):
  1. [WARNING - CRITICAL] "P-4401A has modified bearing housing — do NOT order
     standard replacement housing. Contact Flowserve rep (Jim Davis, 832-555-0147)
     for modified drawing DWG-P4401A-REV3."
     — T. Williams, 2024-12-03 | Verified by R. Chen, 2025-01-15

  2. [TIP] "P-4401A suction valve requires 1-1/4 inch wrench for packing gland —
     not in standard tool kit. Pre-stage before job start."
     — M. Birklett, 2025-08-20 | Verified by J. Hernandez, 2025-08-22

Approved Procedure:
  "Centrifugal Pump Mechanical Seal Replacement — Type 2100" (Version 3)
  Estimated duration: 6.5 hours | 12 steps
  Safety: LOTO required, confirm zero energy, hydrocarbon drain and flush
[/PLANT MEMORY]
```

LIAM responds to Sarah:

> "Here's what you need to know about seal replacement on P-4401A. There are several critical notes from experienced planners:
>
> **CRITICAL WARNING:** P-4401A has a modified bearing housing. Do not order standard replacement parts. Contact the Flowserve rep (Jim Davis, 832-555-0147) for the modified drawing.
>
> **Seal selection:** Use John Crane Type 2100 only. The 1600 series does not fit the modified stuffing box. Also, the bore was machined in 2019, so you need the oversized seal seat — confirm seat OD before ordering.
>
> **After installation:** Use laser alignment for the coupling, not dial indicators. There is a known soft foot issue on the outboard bearing housing.
>
> **Tool note:** Pre-stage a 1-1/4 inch wrench for the suction valve packing gland — it is not in the standard tool kit.
>
> I also have a verified 12-step procedure for this seal type. Want me to pull up the full procedure with time estimates?"

**Why this matters:** Sarah is a new planner. She has never worked on P-4401A before. Without Plant Memory, she would have created a generic job plan, possibly ordered the wrong seal, and the crew would have discovered the problems in the field — wasting hours and potentially damaging equipment. With Plant Memory, every lesson that Mike, Jorge, and Terrence learned over the past two years is available to her instantly.

### P6 Intelligence — User Memory Example

**Scenario:** A turnaround scheduler, David, opens P6 Intelligence to check on the CDU turnaround schedule.

The `GetUserContext` flow returns:

```
[USER CONTEXT]
Name: David Chen
Role: Turnaround Scheduler | Department: Turnaround Planning | Plant: Valero Norco
Top equipment classes: CDU, Reformer, Alkylation Unit
Session count: 52 | Last active: 2026-03-04

Recent queries (last 7 days):
  1. "CDU critical path status" (2026-03-04)
  2. "show me activities behind schedule in CDU" (2026-03-04)
  3. "resource histogram for mechanical crafts week 12" (2026-03-03)
  4. "what-if: delay 100-MECH-1042 by 3 days" (2026-03-02)
  5. "CDU turnaround float analysis" (2026-03-01)

Saved queries (by use count):
  1. "CDU Critical Path Dashboard" (used 44 times)
  2. "Behind Schedule Activities" (used 31 times)
  3. "Resource Histogram — Mechanical" (used 27 times)
[/USER CONTEXT]
```

P6 Intelligence responds:

> "Welcome back, David. Since your last session, two activities on the CDU critical path have changed:
>
> - **100-MECH-1042** (HX-4401 Bundle Pull): Now **2 days behind** original baseline. Current finish: March 18 vs. baseline March 16. This was the activity you ran a what-if scenario on last week.
> - **100-ELEC-2015** (CDU Substation Energize): Float reduced from 4 days to 1 day due to predecessor delays.
>
> Want me to pull up the full critical path, or focus on the activities behind schedule?"

**Why this matters:** David did not have to navigate through P6 screens, set up filters, or remember which activities he was tracking. The bot remembered his focus area and proactively surfaced the changes that matter to him.

### P6 Intelligence — Plant Memory Example

**Scenario:** David asks about the duration estimate for the HX-4401 bundle pull.

David types: "Is 5 days realistic for the HX-4401 bundle pull? That seems short."

P6 Intelligence calls `GetPlantMemory` with EquipmentTag = "HX-4401", EquipmentClass = "Heat Exchanger".

Plant Memory returns historical notes:

```
[PLANT MEMORY]
Equipment: HX-4401 (Shell & Tube Heat Exchanger)

Lessons Learned (2 matches):
  1. [VERIFIED] "HX-4401 bundle pull took 7 days in 2023 turnaround, not the
     planned 5. Channel head bolts were severely corroded — added 1.5 days for
     bolt extraction. Budget 2 extra days for contingency."
     — R. Martinez, 2023-11-20 | Turnaround: VN-CDU-TA-2023

  2. [VERIFIED] "HX-4401 requires 150-ton crane for bundle extraction. The 100-ton
     crane used in 2021 could not clear the pipe rack. Coordinate crane mobilization
     at least 2 weeks in advance."
     — D. Chen, 2023-12-01 | Turnaround: VN-CDU-TA-2023

Equipment Notes (1 match):
  1. [WARNING - CAUTION] "HX-4401 tube bundle weighs approximately 22,000 lbs dry.
     Wet weight after hydrotest can reach 28,000 lbs. Factor into rigging plan."
     — Engineering, 2024-02-15
[/PLANT MEMORY]
```

P6 Intelligence responds:

> "Good instinct — the plant memory says 5 days is optimistic. During the 2023 turnaround, the actual duration was 7 days. The channel head bolts were severely corroded and added 1.5 days for extraction. The recommendation from the 2023 team is to budget 2 extra days for contingency.
>
> Also, make sure the rigging plan calls for the 150-ton crane, not the 100-ton. The 100-ton could not clear the pipe rack in 2021. And the wet bundle weight can reach 28,000 lbs, so the rigging calculations need to account for that.
>
> Want me to update the activity duration to 7 days and show the impact on the critical path?"

---

## 10. Build Roadmap

### Phase 1 — User Memory Foundation (1 week)

**Deliverables:**
- SQL Server database with User Memory tables (UserProfile, UserQueryHistory, UserSavedQueries, UserPreferences)
- All indexes and constraints per Section 4 schema
- Power Automate flow: Session Start — Load User Context (Flow 1)
- Power Automate flow: Query Logged (Flow 2)
- Copilot Studio actions: GetUserContext, LogQuery
- Session start topic flow in one agent (LIAM) for validation
- Basic user context injection into system prompt

**Validation criteria:**
- User opens LIAM, context loads in < 2 seconds
- Every query is silently logged
- Second session shows the user's previous queries in context
- New users get a profile created automatically

### Phase 2 — Plant Memory Foundation (1 week)

**Deliverables:**
- Plant Memory tables (PlantLessonsLearned, ApprovedProcedures, PlantGlossary, EquipmentNotes)
- Power Automate flow: Plant Memory Retrieval (Flow 3)
- Power Automate flow: Lesson Learned Capture (Flow 4)
- Copilot Studio actions: GetPlantMemory, SaveLesson
- Seed data: 20-30 sample lessons, notes, and glossary entries for testing
- Equipment tag regex parsing in Copilot agent

**Validation criteria:**
- User asks about specific equipment, relevant plant memory surfaces
- User can add a lesson learned, it appears in the database with IsVerified = 0
- Verified vs. unverified entries are clearly distinguished in bot responses
- Warnings surface with appropriate severity indicators

### Phase 3 — Full Integration (1 week)

**Deliverables:**
- Power Automate flow: Saved Query (Flow 5)
- Copilot Studio action: SaveQuery
- Saved query execution (user says "run [QueryName]" and it executes)
- Plant memory injection alongside SAP/P6 data in LIAM and P6 Intelligence
- Proactive greeting with insights based on user context
- Query topic auto-classification
- Equipment class auto-derivation from query history

**Validation criteria:**
- User can save and run queries by name
- Proactive greeting includes relevant, accurate insights
- Plant memory and user context work together (e.g., user's equipment preferences influence plant memory retrieval ranking)

### Phase 4 — Admin and Verification (2 weeks)

**Deliverables:**
- Verification workflow: Adaptive Cards in Teams for lesson/note approval
- Admin dashboard (Power Apps): user management, plant memory management, verification queue
- Data retention automated flows
- Role-based access enforcement for plant memory writes
- Approved Procedures CRUD interface (for lead planners to create/edit/version procedures)
- Analytics dashboard: query volume, top equipment, lesson capture rate, user adoption

**Validation criteria:**
- Verifier receives Adaptive Card, can approve/reject/edit
- Admin can view all users, all plant memory entries, and manage the verification queue
- Retention flow correctly purges old query history
- Non-planner users cannot add lessons (graceful error message)

### Phase 5 — Optimization and Expansion (Ongoing)

**Deliverables:**
- Full-text search on plant memory (SQL Server FTS or Azure Cognitive Search)
- Cross-agent memory sharing: when a user asks LIAM about equipment, and then opens P6 Intelligence, the P6 agent knows what equipment they were just discussing
- Personalization improvements: frequency-based suggestion ranking, time-of-day patterns, seasonal equipment focus shifts
- Natural language glossary resolution: user says "the big HX" and Copilot automatically resolves to HX-4401 via PlantGlossary
- Bulk import tools for plant memory: import existing lessons from spreadsheets, SharePoint lists, or historical turnaround reports
- Mobile-optimized experience for field planners
- Dataverse migration option (if adopted by the organization)
- API layer for third-party integrations

---

## 11. The Big Picture — Why This Matters

The industrial workforce is in the middle of a generational transition that has no precedent in the modern era.

The numbers are straightforward. In petrochemical refining, power generation, mining, manufacturing, and heavy industry, 30-50% of the experienced workforce will retire within the next 10 years. These are the maintenance planners who have spent 20-40 years learning the quirks of specific equipment. The schedulers who know that the CDU turnaround always takes 2 days longer than planned because of the corroded bolts on HX-4401. The operators who know that when the vibration on K-2201 hits 4.2 mils, it is time to call the bearing crew — not at 5.0 mils when the alarm trips, because by then you are in unplanned shutdown territory.

This knowledge is not in the CMMS. It is not in the P&IDs. It is not in the OEM manuals. It lives in the heads of people who are walking out the door.

The traditional response to this problem has been:

- **Documentation initiatives:** "Let's get everyone to write down what they know." This fails because experienced practitioners are busy doing their jobs, documentation is tedious, and the resulting documents are rarely structured in a way that is searchable or actionable.
- **Mentoring programs:** Valuable but not scalable. One-to-one knowledge transfer takes years, and not every retiree has an apprentice shadowing them.
- **SharePoint libraries:** Where knowledge goes to die. Documents get uploaded, never tagged properly, never updated, and never found by the person who needs them at 2 AM when the pump is down.

Plant Memory is a different approach. It works because:

1. **Capture is frictionless.** One sentence, entered during normal work. The planner just finished a job and the bot asks: "Anything worth saving?" Ten seconds. Done.

2. **Retrieval is automatic.** The planner does not have to search for knowledge. When they ask about a piece of equipment, the relevant lessons surface automatically. The knowledge finds the person who needs it.

3. **It gets richer over time.** Every interaction is an opportunity to capture something. Every question is an opportunity to retrieve something. The system compounds. After one year, a plant might have 500 lessons learned. After five years, 5,000. Each one is a piece of institutional knowledge that would otherwise have been lost.

4. **It is verified.** The verification workflow prevents garbage accumulation. Lead planners review entries before they become official. Bad entries get filtered out. Good entries get endorsed.

5. **It is alive.** This is not a static document library. It is a database that is queried hundreds of times a day, that surfaces relevant knowledge in context, that tracks which entries are most useful, and that grows with every shift.

The Copilot Memory Layer is not just a personalization feature. It is an institutional knowledge preservation system disguised as a chatbot feature. The user-facing benefit — "the bot remembers me" — is the hook. The organizational benefit — "the bot remembers everything everyone has ever learned about this equipment" — is the strategic value.

When the last planner who knows about the modified bore on P-4401A retires, that knowledge does not leave with them. It is in Plant Memory. The next planner who works on that pump will have it surfaced automatically, without even knowing to ask.

That is the big picture. That is why this matters. And that is what we are building.

---

**Document version:** 1.0
**Last updated:** 2026-03-06
**Next review:** 2026-04-06
**Owner:** BAITEKS Engineering
