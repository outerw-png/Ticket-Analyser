# OutSystems Mentor Prompts — Ticket Impact Analyzer (ODC)

> **Status: updated to match the as-built app.** The prompts below include the fixes found while building and testing (see section 17 of `ticket-impact-analyzer-design.md` for the full list of lessons). The seed prompt text is in `prompt-template-seed.md`.

How to use: open ODC Studio, create the app named in each step, open Mentor, paste the prompt, review what it generates, accept, then publish before moving on. Work in order; later prompts assume earlier ones exist. Mentor output varies, so always check it against `ticket-impact-analyzer-design.md` and fix by follow-up prompt ("also add…", "rename…"). Mentor is best at entities, screens and simple logic. Build the REST connector and secrets by hand where noted.

---

## Step 1 — App `TIA_Core`: reference (static) entities

```
Create these static entities, each with Id (Integer), Label (Text 50), Order (Integer), IsActive (Boolean), and these records:
- TicketType: Incident, Change, ServiceRequest, Problem
- TicketStatus: Draft, Submitted, Analyzing, Analyzed, Failed, Closed
- AnalysisStatus: Queued, Running, Completed, Failed, Superseded
- ReviewStatus: Draft, Edited, Approved, Rejected
- RiskLevel: Low, Medium, High, Critical
- Urgency: Low, Normal, High, Immediate
- Priority: P4, P3, P2, P1
- ActionType: Immediate, ShortTerm, LongTerm, Preventive
- ActionStatus: Open, InProgress, Done, Dismissed
- StakeholderType: Business, Technical, Management, External
Also create a normal (non-static) entity Platform with Id, Label, IsActive and records: OutSystems, SAP S/4HANA Public Cloud, REST API, Integration Service, Azure, Business Process, Other.
```

## Step 2 — `TIA_Core`: ticket and analysis entities

```
Create entity Ticket with attributes: ExternalReference (Text 50), Title (Text 250, mandatory), Description (Text 8000, mandatory), ErrorMessages (Text 8000), BusinessImpact (Text 4000), AdditionalNotes (Text 4000), TicketTypeId (TicketType Identifier), PlatformId (Platform Identifier), UserPriorityId (Priority Identifier), StatusId (TicketStatus Identifier), LatestAnalysisId (Long Integer), OwnerUserId (User Identifier), IsDeleted (Boolean), CreatedOn (DateTime), UpdatedOn (DateTime).

Create entity Analysis with: TicketId (Ticket Identifier, indexed), VersionNo (Integer), StatusId (AnalysisStatus Identifier), ReviewStatusId (ReviewStatus Identifier), ExecutiveSummary (Text 4000), TechnicalSummary (Text 8000), ImpactAssessment (Text 8000), SuggestedRootCause (Text 4000), RiskLevelId (RiskLevel Identifier), RiskRationale (Text 2000), UrgencyId (Urgency Identifier), AiUrgencyId (Urgency Identifier), ConfidenceScore (Decimal), PromptTemplateId (Long Integer), RawResponseJson (Text 100000), TriggeredBy (User Identifier), CompletedOn (DateTime), IsEditedByHuman (Boolean), ErrorCode (Text 30), NextAttemptOn (DateTime), AttemptCount (Integer), CreatedOn (DateTime).

Add an index on Ticket(OwnerUserId, StatusId) and Analysis(TicketId, VersionNo).
```

## Step 3 — `TIA_Core`: child entities

```
Create these entities, each with AnalysisId (Analysis Identifier, delete rule Delete) and CreatedOn:
- AffectedComponent: ComponentName (Text 200), ComponentType (Text 50), PlatformId (Platform Identifier), ImpactDescription (Text 1000), ImpactLevelId (RiskLevel Identifier), Confidence (Decimal)
- Stakeholder: RoleName (Text 100), Department (Text 100), StakeholderTypeId (StakeholderType Identifier), Reason (Text 500), NotificationNeeded (Boolean)
- ActionItem: also TicketId (Ticket Identifier); Description (Text 1000), ActionTypeId (ActionType Identifier), Sequence (Integer), SuggestedOwnerRole (Text 100), AssignedToUserId (User Identifier), DueDate (Date), StatusId (ActionStatus Identifier), EffortEstimate (Text 50)
- InvestigationQuestion: Question (Text 1000), Rationale (Text 500), Sequence (Integer), Answer (Text 2000), AnsweredBy (User Identifier), AnsweredOn (DateTime)
- RecommendedSolution: Title (Text 200), Description (Text 4000), Effort (Text 50), ChangeRiskId (RiskLevel Identifier), IsPreferred (Boolean)
- AnalysisFeedback: Rating (Integer), Comment (Text 2000), SectionName (Text 50)
```

## Step 4 — `TIA_Core`: config, prompt and log entities

```
Create entity PromptTemplate: Code (Text 50), VersionNo (Integer), Name (Text 100), SystemPrompt (Text 20000), UserPromptTemplate (Text 20000), OutputSchemaJson (Text 20000), Model (Text 100), Effort (Text 10), MaxOutputTokens (Integer), IsActive (Boolean), Notes (Text 500), with a unique index on (Code, VersionNo).
Create entity AiCallLog: AnalysisId (Long Integer), PromptTemplateId (Long Integer), Provider (Text 30), Model (Text 100), StartedOn (DateTime), DurationMs (Integer), HttpStatus (Integer), PromptTokens (Integer), CompletionTokens (Integer), Attempt (Integer), Succeeded (Boolean), ErrorCode (Text 30), ErrorMessage (Text 2000), CorrelationId (Text 50).
Create entity ErrorLog: CorrelationId (Text 50), Module (Text 50), ActionName (Text 100), ErrorCode (Text 30), Message (Text 2000), StackTrace (Text 20000), UserId (User Identifier), ContextJson (Text 4000), CreatedOn (DateTime).
Create entity AuditLog: EntityName (Text 50), EntityId (Long Integer), EventType (Text 30), UserId (User Identifier), OldValue (Text 4000), NewValue (Text 4000), OccurredOn (DateTime).
Create entity AppSetting: Key (Text 100, unique), Value (Text 1000), Description (Text 250), with records EnableAutoAnalysis=true, LogPayloads=false, MaxAnalysesPerUserPerDay=20, MaxInputChars=24000.
```

## Step 5 — `TIA_Core`: roles

```
Create roles TIA_Consultant, TIA_Reviewer, TIA_Manager, TIA_PromptAdmin, TIA_Admin.
```

## Step 6 — `TIA_Common` library: helpers

```
Create these server actions (public) in this library:
1. Guid_NewCorrelation: no input; output CorrelationId (Text) = a new GUID converted to text.
2. Text_Truncate: inputs Text, MaxLength (Integer); output Result, WasTruncated (Boolean). Cut at MaxLength and append " [...truncated...]" when needed.
3. Sanitizer_Mask: input Text; output MaskedText. Using regular expressions with SINGLE backslashes (backslash is not an escape character in OutSystems text literals; doubled backslashes cause "Unrecognized grouping construct"), replace, in this order, each match with the text shown:
   - email: [A-Za-z0-9._%+\-]+@[A-Za-z0-9.\-]+\.[A-Za-z]{2,} -> [EMAIL]
   - phone (requires a separator, so plain numbers such as error codes are kept): (?<!\w)(\+\d{1,3}[\s.\-]?)?\(?\d{2,4}\)?[\s.\-]\d{3,4}[\s.\-]?\d{3,4}(?!\w) -> [PHONE]
   - IBAN: \b[A-Z]{2}\d{2}[A-Z0-9]{11,30}\b -> [IBAN]
   - IPv4: \b(?:\d{1,3}\.){3}\d{1,3}\b -> [IP]
   - secrets: (?i)\b(password|pwd|token|api[_-]?key|secret)\s*[=:]\s*\S+ -> [REDACTED]
4. Error_Create: inputs Code, Message, Context (Text); output a structure AppError(Code, Message, Context, CorrelationId).
Create a structure ResultStatus (Success Boolean, ErrorCode Text, ErrorMessage Text, CorrelationId Text).
```

## Step 7 — `TIA_Prompts` library: structures and parsing

(Create the `AnalysisResult` structure by hand: right-click **Structures** and use the JSON-to-structure option with the sample in Appendix A of the design doc, including `recommended_solutions`. Rename the root `AnalysisResult`, check the data types (decimals, integers, booleans) and keep the JSON property names. Then use Mentor for the rest.)

```
Create a server action Prompt_Render with inputs Template (Text) and a list of KeyValue (Key, Value Text); output RenderedText. It replaces every occurrence of "{{Key}}" in Template with the Value. Before replacing, escape any "</ticket>" or "<ticket>" found inside Value by replacing it with "[ticket-tag]".

Create a server action Output_Parse with input JsonText and output AnalysisResult, plus IsValid and ErrorMessage (include the parser's error message and position, never the ticket text). First strip leading/trailing markdown code fences (``` or ```json) and whitespace, then deserialize into the AnalysisResult structure. If deserializing fails, return IsValid = False and the error message.

Create a server action Output_Validate with input AnalysisResult and output Issues (list of Text). Add an issue when executive_summary, technical_summary or impact_assessment is empty; when risk_level is not one of Low, Medium, High, Critical; when urgency is not one of Low, Normal, High, Immediate; or when recommended_actions is empty.

Create a server action Budget_Truncate with inputs Title, Description, ErrorMessages, BusinessImpact, AdditionalNotes (Text) and MaxChars (Integer). Outputs: Title, Description, ErrorMessages, BusinessImpact, AdditionalNotes (Text), WasTruncated (Boolean), Warning (Text). If the combined length of the five inputs is at most MaxChars, return them unchanged with WasTruncated False and Warning empty. Otherwise repeatedly shorten the longest of Description, ErrorMessages, BusinessImpact and AdditionalNotes (cut from the end and append " [...truncated...]") until the combined length including Title fits MaxChars. Never shorten Title. Set WasTruncated True and Warning to "Input truncated: " followed by the comma-separated names of the shortened fields.

(Analysis_Execute must use the five returned texts, not the originals. Add TruncationWarning (Text 500) to the Analysis entity and save Warning there.)
```

## Step 8 — `TIA_AI_Connector` library (REST by hand, wrapper by Mentor)

**By hand:**
1. Add secret settings `Claude_ApiKey` (Text, **Is Secret = Yes**) and `Claude_Model` (Text, e.g. `claude-opus-5-5`); set the values in the ODC Portal per stage. Get the key from the Anthropic Console.
2. *Consume REST API* → `POST https://api.anthropic.com/v1/messages`. Paste these samples so structures are generated.

Request:
```json
{"model":"claude-opus-5-5","max_tokens":4000,"system":"You are a helpful assistant.",
 "messages":[{"role":"user","content":"Say hello."}]}
```
Response:
```json
{"id":"msg_123","type":"message","role":"assistant","model":"claude-opus-5-5",
 "content":[{"type":"text","text":"{\"hello\":\"world\"}"}],
 "stop_reason":"end_turn","usage":{"input_tokens":20,"output_tokens":10}}
```
Name the method `CreateMessage`. If you want schema-constrained JSON, also add `output_config` (`format` of type `json_schema` with a `schema`) to the request structure; verify the exact shape in Anthropic's API docs.

Then give Mentor:

```
I have a consumed REST API method CreateMessage (POST https://api.anthropic.com/v1/messages) and two secret settings: Claude_ApiKey and Claude_Model.

In the OnBeforeRequest callback, add the headers "x-api-key" (value of Claude_ApiKey), "anthropic-version" = "2023-06-01" and "content-type" = "application/json". Never log the key or the request headers.

Create a public server action AI_GenerateStructuredCompletion with inputs SystemPrompt, UserPrompt, SchemaJson (Text), Model (Text, use Claude_Model if empty), Effort (Text), MaxTokens (Integer), TimeoutSec (Integer), CorrelationId (Text), MaxRetries (Integer, default 2). Output structure AiResponse: Content, PromptTokens, CompletionTokens, HttpStatus, DurationMs, Succeeded, ErrorCode, ErrorMessage, StopReason.

Build the request with system = SystemPrompt and one user message = UserPrompt; do not send temperature, top_p, top_k or a thinking parameter (rejected with HTTP 400 on current models). If Effort is not empty set output_config.effort. If SchemaJson is not empty, include output_config.format of type json_schema with that schema. Call CreateMessage. Fill token counts from usage.input_tokens and usage.output_tokens, and StopReason from stop_reason.

After the call, check in THIS order: (1) HTTP failure; (2) stop_reason "max_tokens" -> Succeeded = False, ErrorCode "AI-006" "Output was cut off; increase MaxTokens", even when some text came back, and never retry; (3) stop_reason "refusal" -> ErrorCode "AI-005"; (4) set Content to the concatenated text of ALL content blocks whose type is "text" (ignore thinking blocks; do not use the first block); (5) empty Content -> ErrorCode "AI-007". Retry up to MaxRetries on HTTP 408, 429, 529 and 5xx with 2 and then 6 seconds of waiting; no retry on 400, 401, 403, AI-005, AI-006 or AI-007. Map timeout to AI-001, 429/529 to AI-002, 5xx to AI-003, 401/403 to AI-004. When the HTTP status is not 2xx, put the first 500 characters of the response body (Anthropic's error message) into ErrorMessage, prefixed with the status; never include headers, the API key or the prompt. Never throw; always return AiResponse.

Also create AI_TestConnection that sends "Reply with OK" and returns Success, LatencyMs, ErrorMessage.
```

Troubleshooting: 401 = wrong key; 400 naming a field = remove or fix that parameter (also an empty Model or an empty prompt); 404 = wrong model ID.

Also: ODC's debugger does not step into another module, so inspect AiResponse after the call or read AiCallLog. If the response text shows garbled characters (for example `â` for an arrow), the response is being decoded as Latin-1: make the connector decode the body as UTF-8 (and send `Accept-Charset: utf-8`).

---

## Step 9 — `TIA_Core`: submit and queue

```
Create a Service Action Ticket_Submit with input TicketId (Ticket Identifier) and output Success, ErrorCode, ErrorMessage, AnalysisId. Steps: get the ticket and check that the current user owns it; validate Title is not empty, Description has at least 30 characters; count today's analyses by this user and fail with QTA-001 if it exceeds the AppSetting MaxAnalysesPerUserPerDay; compute VersionNo as the highest existing VersionNo for the ticket plus 1; create an Analysis with Status Queued and VersionNo; set the Ticket status to Submitted; set ReviewStatusId = ReviewStatus.Draft on the new Analysis (and leave RiskLevelId, UrgencyId, AiUrgencyId empty; those attributes must NOT be mandatory); set Ticket.LatestAnalysisId; write an AuditLog record with EventType "Submitted"; commit the transaction; then wake the timer ProcessAnalysisQueue. Ticket_Submit must never call Analysis_Execute or the AI itself (the user would wait for the whole AI call and the request is cancelled).

Create a timer ProcessAnalysisQueue. Select up to 5 Analysis records with Status Queued (or Failed with ErrorCode starting AI-001, AI-002, AI-003, AttemptCount below 3 and NextAttemptOn in the past), oldest first. For each one: set its Status to Running, increment AttemptCount and commit, call Analysis_Execute, and handle exceptions per item so one failure does not stop the rest. If more queued items remain at the end, wake the timer again.
```

## Step 10 — `TIA_Core`: execute analysis

```
Create a server action Analysis_Execute with input AnalysisId. Steps:
1. Load the Analysis, its Ticket, and the active PromptTemplate with Code = "ANALYZE_TICKET".
2. Read AppSetting MaxInputChars. Apply Sanitizer_Mask to Description, ErrorMessages, BusinessImpact and AdditionalNotes, then Budget_Truncate.
3. Build the system prompt from the template and the user prompt with Prompt_Render using the placeholders Title, Description, ErrorMessages, BusinessImpact, AdditionalNotes, TicketType, UserPriority, Platform, OutputLanguage ("English").
1b. If no active template is found, fail the analysis with CFG-001 "No active prompt template ANALYZE_TICKET", write an ErrorLog and do not call the AI. Store the template Id in Analysis.PromptTemplateId and AiCallLog.PromptTemplateId.
4. Generate a CorrelationId and call AI_GenerateStructuredCompletion using the template Model, Effort and MaxOutputTokens (16000). Use the five texts returned by Budget_Truncate (not the originals) for the prompt, and save its Warning in Analysis.TruncationWarning.
5. Create an AiCallLog record from the response, whether it succeeded or not.
6. If it failed, set Analysis Status to Failed with the ErrorCode, set NextAttemptOn (+2 minutes for AI-001/002/003), write an ErrorLog and stop.
7. Call Output_Parse (a deserialization failure must not raise an unhandled exception). If invalid, make ONE repair call with the instruction "The following JSON is incomplete or invalid. Return only the corrected complete JSON." followed by the text, then parse again. If the connector returned AI-006, do not repair. If still invalid, mark the Analysis Failed with AI-010, write an ErrorLog with the parse error message and position (no ticket text) and end normally. Both parse paths (first parse and repair parse) must converge on one shared persist step, so child records are saved either way.
8. Call Output_Validate. Coerce unknown risk levels to Medium.
9. Calculate the final Urgency as the higher of the AI urgency and a rule floor: if the ticket priority is P1 or the text contains "production down", "data loss", "security" or "month-end", the floor is High; if the risk is Critical the floor is Immediate. Save AiUrgencyId and UrgencyId.
9b. Map Claude's text values to the static entity records, case-insensitively and ignoring spaces, underscores and hyphens, with defaults: risk_level and impact_level and change_risk -> RiskLevel (default Medium), urgency -> Urgency (default Normal), stakeholder type -> StakeholderType (default Business), action type -> ActionType (default ShortTerm).
10. In one transaction, save the summaries, risk, urgency, ConfidenceScore, RawResponseJson, create AffectedComponent, Stakeholder, ActionItem (status Open, ActionTypeId and SuggestedOwnerRole from the answer, AssignedToUserId and DueDate empty), InvestigationQuestion and RecommendedSolution records from the lists, set the Analysis to Completed, mark the previous Analysis of this ticket as Superseded, update Ticket.LatestAnalysisId and Ticket status to Analyzed, and write an AuditLog record.
Wrap everything in exception handlers: any unexpected exception sets the Analysis to Failed with SYS-999 and writes an ErrorLog.
```

## Step 11 — `TIA_Core`: service actions for the UI

```
Create these Service Actions, each checking that the current user owns the ticket or has role TIA_Reviewer, TIA_Manager or TIA_Admin, and returning Success, ErrorCode, ErrorMessage:
- Ticket_Save (create or update a Ticket as Draft, validating mandatory fields)
- Ticket_List (inputs: Search text, StatusId, TicketTypeId, RiskLevelId, StartIndex, MaxRecords; output: list with latest risk and urgency and a total count; non-managers only see their own tickets)
- Ticket_Delete (soft delete, owner or admin only)
- Analysis_GetStatus (input AnalysisId; output StatusId, ErrorCode)
- Analysis_Get (input AnalysisId; output the Analysis with its components, stakeholders, actions, questions and solutions)
- Analysis_UpdateSection (inputs AnalysisId, SectionName, NewText; saves the edit, sets IsEditedByHuman and ReviewStatus Edited, writes an AuditLog with old and new value)
- Analysis_Approve and Analysis_Reject (role TIA_Reviewer or TIA_Admin; Reject requires a reason)
- Question_SaveAnswer, Feedback_Submit, Action_Update
- Dashboard_GetKpis (open tickets, analyses this week, count of High and Critical risks, open action items for the current user)
- Ticket_Get (input TicketId; output the Ticket; owner or elevated role; soft-deleted tickets are "not found")
- Action_List (inputs StatusId, TicketId, MineOnly, StartIndex, MaxRecords; outputs TotalCount and a list of the public structure ActionItemListItem; visibility like Ticket_List)
```

Notes: Analysis_Get's AnalysisDetail must also carry TicketTitle, TruncationWarning and AiUrgencyId, and Ticket_List's TicketListItem the latest analysis id. Make the static entities AnalysisStatus, ReviewStatus, RiskLevel, Urgency, TicketType, TicketStatus, Priority, Platform, ActionType, ActionStatus and StakeholderType, and the entities AffectedComponent, Stakeholder, ActionItem, InvestigationQuestion, RecommendedSolution, Ticket, Analysis, AppSetting, PromptTemplate **Public** (read-only exposure if available) so `TIA_Web` and `TIA_Admin` can use them; refresh the dependencies (Ctrl+Q) after every change in `TIA_Core`.

## Step 12 — App `TIA_Web`: screens (one prompt per screen)

Mentor can create the app skeleton and screens but **cannot** add module dependencies, consume Service Actions or configure the layout menu and exception handler: do those by hand (Ctrl+Q for dependencies). Start every screen prompt with: *"Use only the TIA_Core Service Actions X and Y and the TIA_Core public static entities Z. Do not create local entities, do not access any entity directly, and do not generate list/detail/edit scaffolding."* Mentor otherwise generates generic CRUD screens on its own entities (wrong data model, no role checks).

After every screen: check the widget tree for leftover empty containers and that headers and content items of Tabs pair one-to-one. Publish and test with a real ticket before the next screen.

### 12a Layout
```
In TIA_Web set the main layout menu to Dashboard, New Ticket, My Tickets, Action Items. Add an OnException handler on the layout: AccessDenied goes to an AccessDenied screen, other exceptions show a generic Feedback Message.
```

### 12b NewTicket (edit draft and copy mode included)
```
Create a screen NewTicket with optional inputs TicketId and CopyFromTicketId. Use only Ticket_Save, Ticket_Submit, Ticket_Get and the TIA_Core static entities TicketType, Priority and Platform. Form: Title, TicketType, Priority, Platform dropdowns and text areas Description, ErrorMessages, BusinessImpact, AdditionalNotes with character counters. Buttons "Save draft" (Ticket_Save) and "Analyze" (Ticket_Save, then Ticket_Submit, then navigate to AnalysisResult with TicketId and AnalysisId). After a successful save store the returned TicketId in TicketVar.Id so a second click updates the same ticket. Send only user-entered fields (never Status, Owner, IsDeleted, LatestAnalysisId, CreatedOn). If TicketId is given: load with Ticket_Get; only Drafts are editable (otherwise open as a copy). If CopyFromTicketId is given: load with Ticket_Get, leave the Id empty, prefix the title with "Copy of ". Show a loading placeholder while loading and every ErrorMessage when Success is false. Validate Title 5-250 characters and Description at least 30.
```

### 12c AnalysisResult — part A (state and polling)
```
Replace the placeholder content of AnalysisResult (inputs TicketId, AnalysisId). Use only Analysis_Get, Analysis_GetStatus, Ticket_Submit and the TIA_Core structures. The three states are driven ONLY by GetAnalysis (never by a local StatusId variable and never by TextToIdentifier): outer If "not GetAnalysis.IsDataFetched" -> neutral skeleton; inside its False branch: If status is Queued or Running (typed AnalysisStatus static records) -> progress card; False branch -> If status is Failed -> failed card with Retry (Ticket_Submit); False branch -> the completed view. These Ifs must be NESTED, not siblings. Polling: in OnReady always start a 3-second interval clicking a hidden PollButton bound to PollStatus; PollStatus calls Analysis_GetStatus, and when the status is Completed or Failed it clears the interval and refreshes GetAnalysis; also clear it after 100 polls and in OnDestroy. If AnalysisId is 0 show "This ticket has not been analysed yet" with a link to NewTicket (edit) and MyTickets. The header shows TicketTitle, the executive summary as normal text, and tags "Risk: <label>", "Urgency: <label>", "Review: <label>" (Label of the static records, coloured), plus confidence as a percentage.
```
Make `AnalysisStatus` public and refresh the dependency first, otherwise Mentor falls back to `TextToIdentifier`, which never matches.

### 12d AnalysisResult — part B (tabs and actions)
```
Below the header add Tabs with exactly nine headers/content items, in this order: Summary, Technical, Impact, Cause (text with Edit/Save/Cancel through Analysis_UpdateSection with section names ExecutiveSummary, TechnicalSummary, ImpactAssessment, SuggestedRootCause), Components (table), Stakeholders (table with Role, Department, Type, Reason, Notify), Risk (tags, rationale, "AI proposed" urgency only when AiUrgencyId differs from UrgencyId), Actions (table with a status dropdown calling Action_Update; pass AssignedToUserId and DueDate through unchanged, empty is allowed), Questions (answer text area and Save calling Question_SaveAnswer); optionally a tenth tab Solutions (RecommendedSolutions as cards). Under the tabs: an info box with TruncationWarning when not empty; Approve and Reject (reason popup) buttons for the roles TIA_Reviewer and TIA_Admin; a feedback card (1-5 and a comment; Feedback_Submit with section "Overall"). Keep the selected tab after refreshes, disable buttons while a call runs and show every ErrorMessage. Use proper tables with header rows and Labels of static records instead of numbers.
```

### 12e MyTickets
```
Create MyTickets using only Ticket_List and Ticket_Delete (inspect their real parameters). Filters: search, Status, Type, Risk (each with an "All" option; 300 ms debounce; back to page 1 on change) and Clear, all in one flex row. A paged table (10 per page, total count from Ticket_List) with Id, Title, Type, Status, Risk (coloured tag or "Not assessed"), Urgency, Created on. Row actions: Open (Draft tickets go to NewTicket with TicketId; others to AnalysisResult with TicketId and the latest AnalysisId), Duplicate (NewTicket with CopyFromTicketId), Delete (confirmation popup, Ticket_Delete). Loading and empty states. Use the standard Pagination arrows only.
```

### 12f Dashboard
```
Create Dashboard (first menu item, default screen) using only Dashboard_GetKpis and Ticket_List: four KPI cards (Open tickets, Analyses this week, High and Critical risks (red above 0), Open action items) and a Recent tickets table (10 newest: Id, Title, Status, Risk tag, Created on, Open with the same Draft rule as MyTickets), loading and empty states, and a New Ticket button.
```

### 12g ActionItems (needs Action_List from Step 11)
```
Create ActionItems using only Action_List and Action_Update: filters (Status with "All", "Only mine", Clear), a paged table (#, Action, Ticket link, Type label, Owner role, Assigned to or "Unassigned", Due date (red when overdue), Status dropdown, Effort). Changing the status calls Action_Update with the other values passed through. Give the status dropdown a min-width of 150px and put the filter widgets in one flex container with sensible widths.
```

## Step 13 — `TIA_Core` admin actions, then app `TIA_Admin`

### 13a `TIA_Core` actions (publish after each)
```
Prompt_List (Code filter; list item structure PromptTemplateListItem without the long texts, with UsageCount), Prompt_Get (Id), Prompt_Save (validate Code, Name, Model, Effort empty/low/medium/high, MaxOutputTokens 500-32000, prompts required, {{Title}} and {{Description}} present, OutputSchemaJson valid JSON; new version = highest VersionNo + 1 and IsActive False; update refused with PRM-001 when the version has been used by analyses; copy only editable fields; AuditLog with a short summary, never the full prompt text) and Prompt_Activate (refuse an incomplete template with PRM-007; one transaction: deactivate the other versions of the same Code with Advanced SQL `UPDATE {PromptTemplate} SET [IsActive] = @IsActive WHERE [Code] = @Code AND [Id] <> @Id` using a Boolean parameter @IsActive = False, then activate this one). All require TIA_PromptAdmin or TIA_Admin (SEC-002 otherwise).
```
```
Settings_List and Settings_Save (TIA_Admin only; never expose keys or secrets; validate MaxInputChars 1000-200000, MaxAnalysesPerUserPerDay 1-1000, EnableAutoAnalysis and LogPayloads true/false; refuse unknown keys with SET-005; AuditLog old and new value).
```
```
Monitoring_GetUsage, ErrorLog_List, AiCallLog_List, AuditLog_List (TIA_Admin only; DateFrom and DateTo, refuse ranges over 90 days with VAL-001; include the whole last day; paged; newest first; truncate audit values to 200 characters).
```

### 13b `TIA_Admin` app
Create a Reactive Web App, add `TIA_Core` as a dependency, assign yourself `TIA_Admin` and `TIA_PromptAdmin` in the ODC Portal, then one prompt per screen: layout with menu and role-restricted screens (Prompt Templates for TIA_PromptAdmin or TIA_Admin; Settings and Monitoring for TIA_Admin) and an AccessDenied screen; PromptTemplates list and PromptTemplateEdit (honour `AsNewVersion`: hide plain Save and make "Save as new version" primary); Settings (per-row save, test that the correct row is saved, note that secrets live in the ODC Portal); Monitoring (date range, tabs Usage, Errors, AI Calls, Audit, one content item per tab header).

## Step 14 — Hardening

```
In TIA_Web set the screen roles (Consultant, Reviewer, Manager, Admin) with an AccessDenied screen; Managers are read-only on AnalysisResult.
```
```
Create a Timer Housekeeping (daily at 02:00) reading AppSetting RetentionDaysRawResponse = 90, RetentionDaysLogs = 90, RetentionMonthsDeletedTickets = 12: clear RawResponseJson on old analyses, delete old AiCallLog and ErrorLog rows, permanently delete soft-deleted tickets older than the retention with their children, in batches of 500 with a commit per batch, one AuditLog row "Housekeeping" with counts at the end.
```
Then run the regression set (5 tickets) in `ticket-impact-analyzer-design.md` section 17.4.

---

## Seed the prompt template

Use `prompt-template-seed.md` (system prompt, user prompt with the JSON skeleton, `MaxOutputTokens` = 16000, `Effort` = medium, empty schema). Enter it through the `TIA_Admin` PromptTemplateEdit screen once it exists; long texts are awkward to paste in Studio's Edit data grid.

## Tips for Mentor
- One screen or action group per prompt; publish and test after each.
- State the data constraints up front ("use only these Service Actions; no local entities; no generated CRUD"); otherwise Mentor builds on its own entities.
- Mentor cannot add dependencies, configure layout menus or exception handlers, or set secrets: do these by hand.
- If Mentor skips a rule, name the missing rule and the action ("In Analysis_Execute add step 9, the urgency floor").
- Check identifier types, delete rules (children of Analysis cascade), Is Mandatory flags (only keys needed at creation) and entity Public flags after every entity change.
- Always set secrets (API key) by hand in the ODC Portal. Never put them in a prompt or the chat.
- Do not hold debugger breakpoints for long (the request is cancelled: `OS-BERT-00000 The operation was canceled`).
