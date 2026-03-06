# Power Automate Flows

Five flows connect Copilot Studio to the memory layer. Each flow is triggered by an HTTP request from the Copilot agent and returns a structured response.

All flows use **Premium connectors** (SQL Server or Dataverse). Standard licensing does not support these connectors — ensure the appropriate Power Automate plan is in place.

---

## Flow 1 — Session Start: Load User Context

**Purpose:** When a user opens a Copilot conversation, load their profile, recent queries, and top saved queries. Format everything into a context block that gets injected into the system prompt.

**Trigger:** HTTP request (POST) from Copilot Studio
**Connector:** SQL Server connector (or Dataverse connector if using Dataverse)

### Input

| Parameter | Type | Source |
|---|---|---|
| `UserID` | NVARCHAR(128) | M365 Azure AD object ID, resolved by Copilot Studio at conversation start |

### Steps

1. **Query UserProfile**
   - `SELECT * FROM UserProfile WHERE UserID = @UserID AND ActiveFlag = 1`
   - If no record found, INSERT a new UserProfile with default values and continue with the new record.
   - If record found, UPDATE `LastActiveDate = GETDATE()`.

2. **Query last 5 UserQueryHistory**
   - `SELECT TOP 5 QuestionText, TopicCategory, EquipmentReferenced, Timestamp FROM UserQueryHistory WHERE UserID = @UserID ORDER BY Timestamp DESC`

3. **Query top 5 UserSavedQueries by UseCount**
   - `SELECT TOP 5 QueryName, QueryText, UseCount FROM UserSavedQueries WHERE UserID = @UserID AND ActiveFlag = 1 ORDER BY UseCount DESC`

4. **Query UserPreferences**
   - `SELECT PreferenceName, PreferenceValue FROM UserPreferences WHERE UserID = @UserID`

5. **Format context string**
   - Assemble all results into a structured text block using the format below.

### Output

Formatted `[USER CONTEXT]` block returned to Copilot Studio for system prompt injection:

```
[USER CONTEXT]
Name: Mike Reynolds
Role: Turnaround Planner
Department: Maintenance — CDU/VDU
Preferred Equipment: Centrifugal Pumps, Heat Exchangers
Frequent Topics: scheduling, P6 logic, material status

Recent Queries:
1. "What's the status of P-2201A bearing replacement?" (2026-03-05)
2. "Show me all open WOs for CDU heat exchangers" (2026-03-04)
3. "What's the lead time on Cameron Kammprofile gaskets?" (2026-03-04)
4. "P6 logic tie between HX-4401 and V-1105?" (2026-03-03)
5. "Who approved the hot bolt procedure for Unit 4?" (2026-03-01)

Saved Queries:
1. "My unit's open WOs" (used 47 times)
2. "CDU equipment overdue for PM" (used 31 times)
3. "P-2201 bearing history" (used 22 times)
4. "Turnaround critical path activities" (used 18 times)
5. "Material shortages for current TA" (used 12 times)

Preferences:
- DefaultUnit: CDU
- DateFormat: MM/DD/YYYY
- ShowEquipmentHistory: true
[/USER CONTEXT]
```

### Error Handling

- If the database is unreachable, return a minimal context block with just the UserID and a flag indicating context could not be loaded. The Copilot should still function — just without personalization.
- Log all errors to a dedicated error table or Application Insights.

---

## Flow 2 — Query Logged

**Purpose:** After Copilot delivers a response, log the user's question for history and analytics. Runs silently in the background — the user never sees this flow execute.

**Trigger:** HTTP request (POST) from Copilot Studio
**Connector:** SQL Server connector (or Dataverse connector if using Dataverse)

### Input

| Parameter | Type | Required | Description |
|---|---|---|---|
| `UserID` | NVARCHAR(128) | Yes | M365 AAD object ID |
| `QuestionText` | NVARCHAR(2000) | Yes | The user's question, verbatim |
| `TopicCategory` | NVARCHAR(128) | No | Category assigned by the Copilot (e.g., "scheduling", "equipment lookup") |
| `EquipmentReferenced` | NVARCHAR(256) | No | Equipment tag mentioned in the question, if any |

### Steps

1. **INSERT to UserQueryHistory**
   ```sql
   INSERT INTO UserQueryHistory (UserID, QuestionText, TopicCategory, EquipmentReferenced)
   VALUES (@UserID, @QuestionText, @TopicCategory, @EquipmentReferenced)
   ```

2. **UPDATE UserProfile.LastActiveDate**
   ```sql
   UPDATE UserProfile
   SET LastActiveDate = GETDATE()
   WHERE UserID = @UserID
   ```

### Output

| Field | Value |
|---|---|
| `Status` | `"success"` or `"error"` |
| `QueryID` | The identity of the inserted row (returned on success) |

### Notes

- This flow runs **asynchronously**. Copilot does not wait for it to complete before showing the response to the user.
- `TopicCategory` and `EquipmentReferenced` are best-effort fields. The Copilot agent should attempt to classify the question and extract equipment references, but NULL values are acceptable.
- `WasHelpful` is not set at this stage — it may be updated later if the user provides feedback.

---

## Flow 3 — Plant Memory Retrieval

**Purpose:** When a user asks about a specific piece of equipment or failure mode, retrieve relevant lessons learned and equipment notes from Plant Memory.

**Trigger:** HTTP request (POST) from Copilot Studio
**Connector:** SQL Server connector (or Dataverse connector if using Dataverse)

### Input

| Parameter | Type | Required | Description |
|---|---|---|---|
| `EquipmentTag` | NVARCHAR(128) | No* | Specific equipment tag (e.g., "P-2201A") |
| `FailureMode` | NVARCHAR(256) | No* | Failure mode to search for (e.g., "bearing failure") |
| `EquipmentClass` | NVARCHAR(128) | No | Equipment class for broader matching (e.g., "Centrifugal Pump") |

*At least one of `EquipmentTag` or `FailureMode` must be provided.

### Steps

1. **SELECT from PlantLessonsLearned WHERE matching**
   ```sql
   SELECT LessonID, EquipmentClass, EquipmentTag, FailureMode, LessonText,
          SourceWO, AddedBy, AddedDate, VerifiedBy, VerifiedDate, UseCount
   FROM PlantLessonsLearned
   WHERE ActiveFlag = 1
     AND (EquipmentTag = @EquipmentTag OR @EquipmentTag IS NULL)
     AND (FailureMode LIKE '%' + @FailureMode + '%' OR @FailureMode IS NULL)
     AND (EquipmentClass = @EquipmentClass OR @EquipmentClass IS NULL)
   ORDER BY VerifiedDate DESC, UseCount DESC
   ```

2. **SELECT from EquipmentNotes WHERE matching**
   ```sql
   SELECT NoteID, EquipmentTag, NoteText, AddedBy, AddedDate, NoteType
   FROM EquipmentNotes
   WHERE ActiveFlag = 1
     AND EquipmentTag = @EquipmentTag
   ORDER BY
     CASE NoteType WHEN 'warning' THEN 1 WHEN 'tip' THEN 2 WHEN 'history' THEN 3 END,
     AddedDate DESC
   ```

3. **SELECT from ApprovedProcedures WHERE matching** (if EquipmentClass is provided)
   ```sql
   SELECT ProcedureID, EquipmentClass, TaskType, Steps, ApprovedBy, ApprovedDate, Version
   FROM ApprovedProcedures
   WHERE ActiveFlag = 1
     AND EquipmentClass = @EquipmentClass
   ORDER BY Version DESC
   ```

4. **INCREMENT UseCount** on retrieved lessons
   ```sql
   UPDATE PlantLessonsLearned
   SET UseCount = UseCount + 1
   WHERE LessonID IN (@RetrievedLessonIDs)
   ```

5. **Format results** into a structured block for Copilot response injection.

### Output

Formatted plant memory block:

```
[PLANT MEMORY — P-2201A]

⚠ WARNINGS:
- Bearing replacement requires special puller tool T-445 from Tool Room B.
  Standard puller will damage the shaft shoulder. (Added by: J. Martinez, 2025-08-14, VERIFIED by R. Chen)

LESSONS LEARNED:
- Bearing failure on this pump is often preceded by elevated vibration readings
  2-3 weeks before seizure. Check vibration trending before scheduling.
  (Source WO: WO-2025-4412, Added by: J. Martinez, 2025-09-02, VERIFIED by R. Chen)
- Last two bearing failures were caused by contaminated lube oil. Check
  oil sample results before assuming mechanical failure.
  (Added by: S. Patel, 2025-11-20, UNVERIFIED)

TIPS:
- Coupling alignment on P-2201A is tight — use laser alignment, not dial indicators.
  (Added by: T. Williams, 2024-06-10, VERIFIED by R. Chen)

HISTORY:
- Pump was re-rated from 1800 to 2200 GPM in 2019 TA. Original design curves
  no longer apply — use revised curves in SAP document DM-2201A-REV3.
  (Added by: K. O'Brien, 2024-01-15, VERIFIED by R. Chen)

[/PLANT MEMORY]
```

### Notes

- **Warnings are always shown first.** They represent safety-critical or damage-critical information.
- **Unverified lessons are still returned** but clearly marked as `UNVERIFIED`. The Copilot should caveat them in its response.
- `UseCount` tracking enables analytics on which lessons are most valuable.

---

## Flow 4 — Lesson Learned Capture

**Purpose:** Allow users to submit new lessons learned directly through the Copilot conversation. The lesson is stored as unverified until a lead planner or engineer reviews it.

**Trigger:** HTTP request (POST) from Copilot Studio (via action button or conversational prompt)
**Connector:** SQL Server connector (or Dataverse connector if using Dataverse)

### Input

| Parameter | Type | Required | Description |
|---|---|---|---|
| `UserID` | NVARCHAR(128) | Yes | M365 AAD object ID of the submitter |
| `EquipmentTag` | NVARCHAR(128) | No | Specific equipment tag |
| `EquipmentClass` | NVARCHAR(128) | Yes | Equipment class (e.g., "Centrifugal Pump") |
| `FailureMode` | NVARCHAR(256) | No | What failed or what the lesson addresses |
| `LessonText` | NVARCHAR(2000) | Yes | The lesson itself, in plain language |
| `SourceWO` | NVARCHAR(64) | No | Work order number, if applicable |

### Steps

1. **Resolve user name** from UserProfile
   ```sql
   SELECT Name FROM UserProfile WHERE UserID = @UserID
   ```

2. **INSERT to PlantLessonsLearned**
   ```sql
   INSERT INTO PlantLessonsLearned
       (EquipmentClass, EquipmentTag, FailureMode, LessonText, SourceWO, AddedBy, VerifiedBy, VerifiedDate)
   VALUES
       (@EquipmentClass, @EquipmentTag, @FailureMode, @LessonText, @SourceWO, @UserName, NULL, NULL)
   ```
   - `VerifiedBy` is explicitly set to NULL.
   - `VerifiedDate` is explicitly set to NULL.
   - `AddedDate` defaults to GETDATE().
   - `ActiveFlag` defaults to 1.
   - `UseCount` defaults to 0.

3. **Return the new LessonID**
   ```sql
   SELECT SCOPE_IDENTITY() AS LessonID
   ```

### Output

| Field | Value |
|---|---|
| `LessonID` | The identity of the newly inserted lesson |
| `Status` | `"success"` or `"error"` |
| `Message` | `"Lesson captured successfully. It will be marked as UNVERIFIED until reviewed by a lead planner."` |

### Notes

- **The lesson starts as UNVERIFIED.** This is intentional. Wrong information in Plant Memory is worse than no information. The verification workflow is a separate process (manual or via a dedicated review flow) handled by lead planners or reliability engineers.
- The Copilot should confirm back to the user: *"Got it. Your lesson has been saved for [Equipment Tag]. It will be reviewed and verified by a lead planner before being marked as approved."*
- Future enhancement: trigger a notification (email or Teams message) to the designated reviewer when a new unverified lesson is submitted.

---

## Flow 5 — Saved Query

**Purpose:** Allow users to bookmark a query for quick re-use. If a query with the same name already exists for the user, update it and increment the use count.

**Trigger:** HTTP request (POST) from Copilot Studio
**Connector:** SQL Server connector (or Dataverse connector if using Dataverse)

### Input

| Parameter | Type | Required | Description |
|---|---|---|---|
| `UserID` | NVARCHAR(128) | Yes | M365 AAD object ID |
| `QueryName` | NVARCHAR(256) | Yes | User-provided label for the saved query |
| `QueryText` | NVARCHAR(2000) | Yes | The actual query text to save |

### Steps

1. **Check if query with same name exists for user**
   ```sql
   SELECT SavedQueryID, UseCount
   FROM UserSavedQueries
   WHERE UserID = @UserID
     AND QueryName = @QueryName
     AND ActiveFlag = 1
   ```

2. **If exists: UPDATE and increment UseCount**
   ```sql
   UPDATE UserSavedQueries
   SET QueryText = @QueryText,
       UseCount = UseCount + 1
   WHERE SavedQueryID = @ExistingSavedQueryID
   ```

3. **If not exists: INSERT new saved query**
   ```sql
   INSERT INTO UserSavedQueries (UserID, QueryName, QueryText)
   VALUES (@UserID, @QueryName, @QueryText)
   ```
   - `CreatedDate` defaults to GETDATE().
   - `UseCount` defaults to 0.
   - `ActiveFlag` defaults to 1.

4. **Return SavedQueryID**
   - On UPDATE: return the existing `SavedQueryID`.
   - On INSERT: `SELECT SCOPE_IDENTITY() AS SavedQueryID`.

### Output

| Field | Value |
|---|---|
| `SavedQueryID` | The identity of the saved query (new or existing) |
| `Status` | `"success"` or `"error"` |
| `IsNew` | `true` if newly created, `false` if updated |
| `Message` | `"Query saved as 'My unit's open WOs'."` or `"Query 'My unit's open WOs' updated (used 48 times)."` |

### Notes

- The Copilot can prompt users to save a query after they ask something for the second or third time: *"You've asked this before. Want me to save it as a quick query?"*
- Saved queries are loaded at session start (Flow 1) and can be offered as suggestions at the beginning of a conversation.

---

## Connector Summary

| Flow | Primary Connector | Fallback |
|---|---|---|
| Session Start — Load User Context | SQL Server | Dataverse |
| Query Logged | SQL Server | Dataverse |
| Plant Memory Retrieval | SQL Server | Dataverse |
| Lesson Learned Capture | SQL Server | Dataverse |
| Saved Query | SQL Server | Dataverse |

**Choosing between SQL Server and Dataverse connectors:**

- **SQL Server connector** — use when the memory layer runs on a dedicated SQL Server instance (on-prem or Azure SQL). Provides full T-SQL capability, better performance on complex queries, and direct control over indexes and execution plans.
- **Dataverse connector** — use when the organization standardizes on Dataverse or needs tight integration with other Power Platform components (Model-Driven Apps, Power BI, security roles). Provides built-in row-level security, audit logging, and no-code admin views.

Both connectors are Premium. The flows above use SQL syntax but translate directly to Dataverse operations (List Rows, Add Row, Update Row) with equivalent filter expressions.

---

## Security Considerations

- All flows authenticate via the **service account** or **connection reference** configured in the Power Automate environment. End users do not need direct database access.
- `UserID` is always resolved from the M365 authentication token — users cannot impersonate other users.
- Plant Memory write operations (Flow 4) should be restricted to authenticated users only. Read operations (Flow 3) are available to all Copilot users.
- Consider adding **rate limiting** on Flow 2 (Query Logged) to prevent excessive logging in automated testing or abuse scenarios.
