# Database Schema

Complete SQL Server schema for the Copilot Memory Layer. Eight tables across two layers.

All tables use:
- `INT IDENTITY(1,1)` primary keys
- `DEFAULT GETDATE()` on timestamp columns
- `ActiveFlag BIT DEFAULT 1` for soft deletes where applicable
- Foreign keys with explicit naming
- Indexes on frequently queried columns

---

## Layer 1 — User Memory

### UserProfile

Stores per-user identity and preferences. `UserID` is the M365 Azure AD object identifier — the unique, immutable GUID that identifies each user in the Microsoft 365 tenant.

```sql
CREATE TABLE UserProfile (
    ProfileID           INT IDENTITY(1,1) PRIMARY KEY,
    UserID              NVARCHAR(128) NOT NULL,
    Name                NVARCHAR(256) NOT NULL,
    Role                NVARCHAR(128) NULL,
    Department          NVARCHAR(128) NULL,
    PreferredEquipmentClasses NVARCHAR(512) NULL,
    FrequentTopics      NVARCHAR(512) NULL,
    OnboardedDate       DATETIME2 DEFAULT GETDATE(),
    LastActiveDate      DATETIME2 DEFAULT GETDATE(),
    ActiveFlag          BIT DEFAULT 1,

    CONSTRAINT UQ_UserProfile_UserID UNIQUE (UserID)
);

CREATE INDEX IX_UserProfile_UserID ON UserProfile (UserID);
CREATE INDEX IX_UserProfile_Department ON UserProfile (Department);
```

**Notes:**
- `UserID` is NVARCHAR(128) to accommodate AAD object IDs (GUIDs in string form, typically 36 chars) with headroom.
- `PreferredEquipmentClasses` and `FrequentTopics` are comma-separated values. If the list grows complex, consider normalizing into junction tables in a future phase.
- `OnboardedDate` is set once at first login. `LastActiveDate` is updated on every session start.

---

### UserQueryHistory

Automatic log of every question a user asks through the Copilot. Used for continuity (loading recent context) and analytics.

```sql
CREATE TABLE UserQueryHistory (
    QueryID             INT IDENTITY(1,1) PRIMARY KEY,
    UserID              NVARCHAR(128) NOT NULL,
    Timestamp           DATETIME2 DEFAULT GETDATE(),
    QuestionText        NVARCHAR(2000) NOT NULL,
    TopicCategory       NVARCHAR(128) NULL,
    EquipmentReferenced NVARCHAR(256) NULL,
    WasHelpful          BIT NULL,

    CONSTRAINT FK_UserQueryHistory_UserProfile
        FOREIGN KEY (UserID) REFERENCES UserProfile(UserID)
);

CREATE INDEX IX_UserQueryHistory_UserID ON UserQueryHistory (UserID);
CREATE INDEX IX_UserQueryHistory_Timestamp ON UserQueryHistory (Timestamp DESC);
CREATE INDEX IX_UserQueryHistory_TopicCategory ON UserQueryHistory (TopicCategory);
CREATE INDEX IX_UserQueryHistory_EquipmentReferenced ON UserQueryHistory (EquipmentReferenced);
```

**Notes:**
- `WasHelpful` is nullable — feedback is optional. NULL means no feedback given, 1 = helpful, 0 = not helpful.
- `TopicCategory` is a free-text field populated by the Copilot agent (e.g., "scheduling", "materials", "equipment lookup").
- `EquipmentReferenced` captures any equipment tag mentioned in the query for cross-referencing with Plant Memory.

---

### UserSavedQueries

Bookmarked queries that users want to re-run. Ranked by `UseCount` for quick access.

```sql
CREATE TABLE UserSavedQueries (
    SavedQueryID        INT IDENTITY(1,1) PRIMARY KEY,
    UserID              NVARCHAR(128) NOT NULL,
    QueryName           NVARCHAR(256) NOT NULL,
    QueryText           NVARCHAR(2000) NOT NULL,
    CreatedDate         DATETIME2 DEFAULT GETDATE(),
    UseCount            INT DEFAULT 0,
    ActiveFlag          BIT DEFAULT 1,

    CONSTRAINT FK_UserSavedQueries_UserProfile
        FOREIGN KEY (UserID) REFERENCES UserProfile(UserID)
);

CREATE INDEX IX_UserSavedQueries_UserID ON UserSavedQueries (UserID);
CREATE INDEX IX_UserSavedQueries_UseCount ON UserSavedQueries (UserID, UseCount DESC);
```

**Notes:**
- `UseCount` increments each time the saved query is executed. Top queries by use count are loaded at session start.
- `QueryName` is a user-provided label (e.g., "My unit's open WOs", "P-2201 bearing history").

---

### UserPreferences

Generic key-value store for user-specific settings. Flexible schema for preferences that don't warrant their own column.

```sql
CREATE TABLE UserPreferences (
    PreferenceID        INT IDENTITY(1,1) PRIMARY KEY,
    UserID              NVARCHAR(128) NOT NULL,
    PreferenceName      NVARCHAR(128) NOT NULL,
    PreferenceValue     NVARCHAR(512) NULL,

    CONSTRAINT FK_UserPreferences_UserProfile
        FOREIGN KEY (UserID) REFERENCES UserProfile(UserID),
    CONSTRAINT UQ_UserPreferences_UserPref UNIQUE (UserID, PreferenceName)
);

CREATE INDEX IX_UserPreferences_UserID ON UserPreferences (UserID);
```

**Notes:**
- The UNIQUE constraint on (UserID, PreferenceName) prevents duplicate preference keys per user.
- Example entries: `DefaultUnit = "CDU"`, `DateFormat = "MM/DD/YYYY"`, `ShowEquipmentHistory = "true"`.

---

## Layer 2 — Plant Memory

### PlantLessonsLearned

The core institutional knowledge table. Each lesson is tied to a specific equipment class, tag, and/or failure mode. Verification workflow ensures quality.

```sql
CREATE TABLE PlantLessonsLearned (
    LessonID            INT IDENTITY(1,1) PRIMARY KEY,
    EquipmentClass      NVARCHAR(128) NULL,
    EquipmentTag        NVARCHAR(128) NULL,
    FailureMode         NVARCHAR(256) NULL,
    LessonText          NVARCHAR(2000) NOT NULL,
    SourceWO            NVARCHAR(64) NULL,
    AddedBy             NVARCHAR(128) NOT NULL,
    AddedDate           DATETIME2 DEFAULT GETDATE(),
    VerifiedBy          NVARCHAR(128) NULL,
    VerifiedDate        DATETIME2 NULL,
    UseCount            INT DEFAULT 0,
    ActiveFlag          BIT DEFAULT 1
);

CREATE INDEX IX_PlantLessonsLearned_EquipmentTag ON PlantLessonsLearned (EquipmentTag);
CREATE INDEX IX_PlantLessonsLearned_EquipmentClass ON PlantLessonsLearned (EquipmentClass);
CREATE INDEX IX_PlantLessonsLearned_FailureMode ON PlantLessonsLearned (FailureMode);
CREATE INDEX IX_PlantLessonsLearned_AddedBy ON PlantLessonsLearned (AddedBy);
```

**Notes:**
- `EquipmentClass` = broad category (e.g., "Centrifugal Pump", "Heat Exchanger", "Fin Fan Cooler").
- `EquipmentTag` = specific equipment identifier (e.g., "P-2201A", "HX-4401", "V-1105").
- `FailureMode` = what went wrong or what the lesson addresses (e.g., "bearing failure", "gasket leak", "instrument drift").
- `SourceWO` = work order number where the lesson originated. NULL if not tied to a specific WO.
- `VerifiedBy` and `VerifiedDate` are NULL until a lead planner or engineer reviews the lesson. Unverified lessons are still surfaced but flagged.
- `UseCount` tracks how many times this lesson has been retrieved in a response.

---

### ApprovedProcedures

Step-by-step procedures for recurring tasks. Versioned and approved. Steps stored as structured JSON.

```sql
CREATE TABLE ApprovedProcedures (
    ProcedureID         INT IDENTITY(1,1) PRIMARY KEY,
    EquipmentClass      NVARCHAR(128) NOT NULL,
    TaskType            NVARCHAR(256) NOT NULL,
    Steps               NVARCHAR(MAX) NOT NULL,
    ApprovedBy          NVARCHAR(128) NULL,
    ApprovedDate        DATETIME2 NULL,
    Version             INT DEFAULT 1,
    ActiveFlag          BIT DEFAULT 1
);

CREATE INDEX IX_ApprovedProcedures_EquipmentClass ON ApprovedProcedures (EquipmentClass);
CREATE INDEX IX_ApprovedProcedures_TaskType ON ApprovedProcedures (TaskType);
```

**Notes:**
- `Steps` is NVARCHAR(MAX) containing a JSON array of step objects. Example format:

```json
[
    {"step": 1, "action": "Isolate pump per LOTO procedure E-2201", "caution": null},
    {"step": 2, "action": "Drain casing and remove coupling guard", "caution": "Residual pressure — open vent valve first"},
    {"step": 3, "action": "Remove bearing housing bolts (12x M16)", "caution": "Support housing to prevent shaft drop"},
    {"step": 4, "action": "Extract bearing using puller tool T-445 from Tool Room B", "caution": "Do NOT use standard puller — shaft shoulder is non-standard"}
]
```

- `TaskType` describes the work (e.g., "Bearing Replacement", "Gasket Changeout", "Calibration Check").
- `Version` increments when a procedure is updated. Only the latest active version is served by default.

---

### PlantGlossary

Plant-specific terminology. Ensures the bot uses the same language as the crew.

```sql
CREATE TABLE PlantGlossary (
    TermID              INT IDENTITY(1,1) PRIMARY KEY,
    Term                NVARCHAR(256) NOT NULL,
    Definition          NVARCHAR(2000) NOT NULL,
    Context             NVARCHAR(512) NULL,
    AddedBy             NVARCHAR(128) NOT NULL,
    AddedDate           DATETIME2 DEFAULT GETDATE(),
    ActiveFlag          BIT DEFAULT 1
);

CREATE INDEX IX_PlantGlossary_Term ON PlantGlossary (Term);
```

**Notes:**
- `Context` clarifies where the term is used (e.g., "Turnaround planning", "CDU operations", "SAP PM module").
- Example entries:
  - Term: "LOTO", Definition: "Lock Out / Tag Out — energy isolation procedure required before maintenance work", Context: "Safety / Maintenance"
  - Term: "Hot bolt", Definition: "Tightening or replacing bolts on flanged connections while the unit is running under pressure and temperature", Context: "Turnaround avoidance"
  - Term: "Skin temp", Definition: "External surface temperature measurement on a vessel or pipe, typically taken with IR gun or thermocouple", Context: "Inspection / Operations"

---

### EquipmentNotes

Per-equipment tribal knowledge. Tips, warnings, and historical context that doesn't fit neatly into a lesson or procedure.

```sql
CREATE TABLE EquipmentNotes (
    NoteID              INT IDENTITY(1,1) PRIMARY KEY,
    EquipmentTag        NVARCHAR(128) NOT NULL,
    NoteText            NVARCHAR(2000) NOT NULL,
    AddedBy             NVARCHAR(128) NOT NULL,
    AddedDate           DATETIME2 DEFAULT GETDATE(),
    NoteType            NVARCHAR(32) NOT NULL DEFAULT 'tip',
    ActiveFlag          BIT DEFAULT 1,

    CONSTRAINT CK_EquipmentNotes_NoteType
        CHECK (NoteType IN ('tip', 'warning', 'history'))
);

CREATE INDEX IX_EquipmentNotes_EquipmentTag ON EquipmentNotes (EquipmentTag);
CREATE INDEX IX_EquipmentNotes_NoteType ON EquipmentNotes (NoteType);
```

**Notes:**
- `NoteType` is constrained to three values:
  - `tip` — helpful information for working on or with the equipment
  - `warning` — critical caution that could affect safety or cause damage
  - `history` — historical context about past work, modifications, or known issues
- Warnings are displayed prominently in Copilot responses. Tips and history are included as supporting context.

---

## Dataverse Equivalents

For organizations using Dataverse instead of (or in addition to) SQL Server, each table maps to a Dataverse table. The key differences:

- Dataverse uses its own GUID-based primary keys (not INT IDENTITY).
- Column names use the publisher prefix (e.g., `baiteks_`).
- Relationships are defined as Dataverse lookups rather than SQL foreign keys.
- Choice columns replace CHECK constraints for fields like `NoteType`.

| SQL Server Table | Dataverse Table Name | Dataverse Display Name | Notes |
|---|---|---|---|
| `UserProfile` | `baiteks_userprofile` | User Profile | UserID mapped to M365 User lookup or stored as text |
| `UserQueryHistory` | `baiteks_userqueryhistory` | User Query History | UserID as lookup to User Profile table |
| `UserSavedQueries` | `baiteks_usersavedqueries` | User Saved Queries | UserID as lookup to User Profile table |
| `UserPreferences` | `baiteks_userpreferences` | User Preferences | UserID as lookup to User Profile table |
| `PlantLessonsLearned` | `baiteks_plantlessonslearned` | Plant Lessons Learned | VerifiedBy as lookup to User Profile or plain text |
| `ApprovedProcedures` | `baiteks_approvedprocedures` | Approved Procedures | Steps stored in multiline text column (JSON) |
| `PlantGlossary` | `baiteks_plantglossary` | Plant Glossary | Term indexed via Dataverse search |
| `EquipmentNotes` | `baiteks_equipmentnotes` | Equipment Notes | NoteType as Choice column (tip/warning/history) |

**Dataverse-specific considerations:**

- **Search:** Dataverse Relevance Search should be enabled on `PlantLessonsLearned`, `EquipmentNotes`, and `PlantGlossary` tables to support full-text queries from Copilot.
- **Security:** Dataverse row-level security can restrict Plant Memory edits to authorized roles (lead planners, reliability engineers) while allowing read access for all Copilot users.
- **Audit:** Dataverse built-in auditing captures change history automatically, which supplements the `AddedBy`/`VerifiedBy` fields.
- **Connector:** Power Automate flows use the **Dataverse connector** (not the SQL Server connector) when targeting Dataverse tables. The Dataverse connector supports richer filtering and native relationship traversal.

---

## Schema Diagram (Logical)

```
USER MEMORY                              PLANT MEMORY
───────────                              ────────────

┌──────────────────┐                     ┌────────────────────────┐
│   UserProfile    │                     │  PlantLessonsLearned   │
│──────────────────│                     │────────────────────────│
│ ProfileID (PK)   │                     │ LessonID (PK)          │
│ UserID (UQ)      │──┐                  │ EquipmentClass         │
│ Name             │  │                  │ EquipmentTag           │
│ Role             │  │                  │ FailureMode            │
│ Department       │  │                  │ LessonText             │
│ ...              │  │                  │ SourceWO               │
│ ActiveFlag       │  │                  │ AddedBy                │
└──────────────────┘  │                  │ VerifiedBy             │
                      │                  │ UseCount               │
    ┌─────────────────┤                  │ ActiveFlag             │
    │                 │                  └────────────────────────┘
    │                 │
    ▼                 ▼                  ┌────────────────────────┐
┌──────────────┐ ┌──────────────────┐   │  ApprovedProcedures    │
│UserQueryHist │ │UserSavedQueries  │   │────────────────────────│
│──────────────│ │──────────────────│   │ ProcedureID (PK)       │
│ QueryID (PK) │ │SavedQueryID (PK) │   │ EquipmentClass         │
│ UserID (FK)  │ │ UserID (FK)      │   │ TaskType               │
│ Timestamp    │ │ QueryName        │   │ Steps (JSON)           │
│ QuestionText │ │ QueryText        │   │ ApprovedBy             │
│ TopicCategory│ │ CreatedDate      │   │ Version                │
│ EquipmentRef │ │ UseCount         │   │ ActiveFlag             │
│ WasHelpful   │ │ ActiveFlag       │   └────────────────────────┘
└──────────────┘ └──────────────────┘
                                         ┌────────────────────────┐
    ┌─────────────────┐                  │  PlantGlossary         │
    │                 │                  │────────────────────────│
    ▼                 │                  │ TermID (PK)            │
┌──────────────────┐  │                  │ Term                   │
│UserPreferences   │  │                  │ Definition             │
│──────────────────│  │                  │ Context                │
│ PreferenceID(PK) │  │                  │ AddedBy                │
│ UserID (FK)  ────┘  │                  │ ActiveFlag             │
│ PreferenceName(UQ)  │                  └────────────────────────┘
│ PreferenceValue  │  │
└──────────────────┘  │                  ┌────────────────────────┐
                      │                  │  EquipmentNotes        │
                      │                  │────────────────────────│
                      │                  │ NoteID (PK)            │
                      │                  │ EquipmentTag           │
                      │                  │ NoteText               │
                      │                  │ AddedBy                │
                      │                  │ NoteType               │
                      │                  │ ActiveFlag             │
                      │                  └────────────────────────┘
```

All User Memory tables reference `UserProfile.UserID` via foreign key. Plant Memory tables are standalone — they reference equipment, not users (except `AddedBy` and `VerifiedBy` as attribution fields, not enforced foreign keys).
