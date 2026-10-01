# OutSystems Mentor Prompts — Ticket Impact Analyzer (ODC)

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
Create entity PromptTemplate: Code (Text 50), VersionNo (Integer), Name (Text 100), SystemPrompt (Text 20000), UserPromptTemplate (Text 20000), OutputSchemaJson (Text 20000), Model (Text 100), Temperature (Decimal), MaxOutputTokens (Integer), IsActive (Boolean), Notes (Text 500), with a unique index on (Code, VersionNo).
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
3. Sanitizer_Mask: input Text; output MaskedText. Using regular expressions, replace email addresses with [EMAIL], phone numbers with [PHONE], IBANs with [IBAN], IPv4 addresses with [IP], and strings following "password=", "pwd=", "token=", "apikey=" or "secret=" with [REDACTED].
4. Error_Create: inputs Code, Message, Context (Text); output a structure AppError(Code, Message, Context, CorrelationId).
Create a structure ResultStatus (Success Boolean, ErrorCode Text, ErrorMessage Text, CorrelationId Text).
```

## Step 7 — `TIA_Prompts` library: structures and parsing

(Easier by hand for the JSON structure: *Add JSON to Structure* using Appendix A of the design doc. Then use Mentor for the rest.)

```
Create a server action Prompt_Render with inputs Template (Text) and a list of KeyValue (Key, Value Text); output RenderedText. It replaces every occurrence of "{{Key}}" in Template with the Value. Before replacing, escape any "</ticket>" or "<ticket>" found inside Value by replacing it with "[ticket-tag]".

Create a server action Output_Parse with input JsonText and output AnalysisResult, plus IsValid and ErrorMessage. First strip leading/trailing markdown code fences (``` or ```json) and whitespace, then deserialize into the AnalysisResult structure. If deserializing fails, return IsValid = False and the error message.

Create a server action Output_Validate with input AnalysisResult and output Issues (list of Text). Add an issue when executive_summary, technical_summary or impact_assessment is empty; when risk_level is not one of Low, Medium, High, Critical; when urgency is not one of Low, Normal, High, Immediate; or when recommended_actions is empty.

Create a server action Budget_Truncate with inputs Title, Description, ErrorMessages, BusinessImpact, AdditionalNotes and MaxChars. If the total length exceeds MaxChars, shorten the longest fields first, never shorten Title, and return the shortened fields plus a Warning text.
```

## Step 8 — `TIA_AI_Connector` library (REST by hand, wrapper by Mentor)

**By hand:** *Consume REST API* → POST `{endpoint}/openai/deployments/{deployment}/chat/completions?api-version=…`. Add header `api-key` from a **secret** setting. Paste a sample request/response so structures are generated. Then:

```
Create a server action AI_GenerateStructuredCompletion with inputs SystemPrompt, UserPrompt, SchemaJson, Model, Temperature (Decimal), MaxTokens (Integer), TimeoutSec (Integer), CorrelationId, MaxRetries (Integer). It calls the chat completions REST method I consumed, with response format set to JSON. Output a structure AiResponse: Content, PromptTokens, CompletionTokens, HttpStatus, DurationMs, Succeeded, ErrorCode, ErrorMessage.
Retry up to MaxRetries on HTTP 408, 429 and 5xx with waiting of 2 seconds then 6 seconds (use a loop and a wait mechanism that fits ODC). Do not retry on 400, 401, 403. Map errors: timeout AI-001, 429 AI-002, 5xx AI-003, 401/403 AI-004, content filter AI-005. Never throw; always return the AiResponse.
```

## Step 9 — `TIA_Core`: submit and queue

```
Create a Service Action Ticket_Submit with input TicketId (Ticket Identifier) and output Success, ErrorCode, ErrorMessage, AnalysisId. Steps: get the ticket and check that the current user owns it; validate Title is not empty, Description has at least 30 characters; count today's analyses by this user and fail with QTA-001 if it exceeds the AppSetting MaxAnalysesPerUserPerDay; compute VersionNo as the highest existing VersionNo for the ticket plus 1; create an Analysis with Status Queued and VersionNo; set the Ticket status to Submitted; write an AuditLog record with EventType "Submitted"; commit the transaction; then wake the timer ProcessAnalysisQueue.

Create a timer ProcessAnalysisQueue. Select up to 5 Analysis records with Status Queued (or Failed with ErrorCode starting AI-001, AI-002, AI-003, AttemptCount below 3 and NextAttemptOn in the past), oldest first. For each one: set its Status to Running, increment AttemptCount and commit, call Analysis_Execute, and handle exceptions per item so one failure does not stop the rest. If more queued items remain at the end, wake the timer again.
```

## Step 10 — `TIA_Core`: execute analysis

```
Create a server action Analysis_Execute with input AnalysisId. Steps:
1. Load the Analysis, its Ticket, and the active PromptTemplate with Code = "ANALYZE_TICKET".
2. Read AppSetting MaxInputChars. Apply Sanitizer_Mask to Description, ErrorMessages, BusinessImpact and AdditionalNotes, then Budget_Truncate.
3. Build the system prompt from the template and the user prompt with Prompt_Render using the placeholders Title, Description, ErrorMessages, BusinessImpact, AdditionalNotes, TicketType, UserPriority, Platform, OutputLanguage ("English").
4. Generate a CorrelationId and call AI_GenerateStructuredCompletion using the template Model, Temperature and MaxOutputTokens.
5. Create an AiCallLog record from the response, whether it succeeded or not.
6. If it failed, set Analysis Status to Failed with the ErrorCode, set NextAttemptOn (+2 minutes for AI-001/002/003), write an ErrorLog and stop.
7. Call Output_Parse. If invalid, call the AI once more with a repair prompt that includes the invalid text, then parse again. If still invalid, mark the Analysis Failed with AI-010.
8. Call Output_Validate. Coerce unknown risk levels to Medium.
9. Calculate the final Urgency as the higher of the AI urgency and a rule floor: if the ticket priority is P1 or the text contains "production down", "data loss", "security" or "month-end", the floor is High; if the risk is Critical the floor is Immediate. Save AiUrgencyId and UrgencyId.
10. In one transaction, save the summaries, risk, urgency, ConfidenceScore, RawResponseJson, create AffectedComponent, Stakeholder, ActionItem (status Open), InvestigationQuestion and RecommendedSolution records from the lists, set the Analysis to Completed, mark the previous Analysis of this ticket as Superseded, update Ticket.LatestAnalysisId and Ticket status to Analyzed, and write an AuditLog record.
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
```

## Step 12 — App `TIA_Web`: screens

```
Create a Reactive Web app with a main layout and menu items Dashboard, New Ticket, My Tickets, Action Items. Use OutSystems UI patterns.

Screen Dashboard: four KPI cards from Dashboard_GetKpis, a list of the 10 most recent tickets with status and risk tags, a donut chart of risk distribution, and a New Ticket button.

Screen NewTicket (optional input TicketId): a form with Title, Type dropdown, Priority dropdown, Platform dropdown, and text areas for Description, Error Messages, Business Impact and Additional Notes, each with a character counter. Buttons "Save draft" (Ticket_Save) and "Analyze" (Ticket_Save then Ticket_Submit then navigate to AnalysisResult). Show validation messages and disable the buttons while saving.

Screen MyTickets: search box, filters for status, type and risk, a table with server-side pagination and sorting calling Ticket_List, status and risk shown as colored tags, and a row menu with Open, Duplicate and Delete (with confirmation).

Screen AnalysisResult (inputs TicketId, optional AnalysisId): while the status is Queued or Running show a progress card and re-check Analysis_GetStatus every 3 seconds until Completed or Failed (stop after 5 minutes with a message). When Completed show a header with title, risk badge, urgency badge and confidence, then Tabs for Executive Summary, Technical Summary, Impact Assessment, Affected Components, Stakeholders, Risk and Urgency, Root Cause, Recommended Actions, Investigation Questions. Each text section has an Edit button that saves through Analysis_UpdateSection. Footer buttons: Re-analyze, Approve, Reject, Copy management summary. Include a 1–5 star rating with comment sending Feedback_Submit. When Failed show the friendly error and a Retry button.

Screen ActionItems: table of ActionItem filtered by status, with inline status change, assignee and due date.
```

## Step 13 — App `TIA_Admin`: admin screens

```
Create a Reactive Web app restricted to role TIA_PromptAdmin and TIA_Admin. Screens:
- PromptTemplates: list grouped by Code with VersionNo and active flag; detail form for SystemPrompt, UserPromptTemplate, OutputSchemaJson, Model, Temperature (0–1), MaxOutputTokens; validate that OutputSchemaJson is valid JSON and that the template contains {{Title}} and {{Description}}; buttons Save as new version, Activate (deactivates other versions with the same Code, with confirmation) and Test.
- Settings: edit AppSetting records with a Save button.
- Monitoring: last 100 AiCallLog records with duration, tokens and success, a chart of calls per day, last 100 ErrorLog records with search by CorrelationId, and AuditLog with date filter (maximum 90 days).
```

## Step 14 — Hardening prompts (use as follow-ups)

```
Add an exception handler to every screen that shows a friendly message with the CorrelationId and redirects AccessDenied exceptions to an access-denied page.
```
```
Add a timer Housekeeping that runs nightly: clear RawResponseJson on Analysis records older than 90 days, delete AiCallLog older than 90 days, delete soft-deleted tickets older than 12 months with their children.
```
```
Add role checks: on screens of TIA_Web allow TIA_Consultant, TIA_Reviewer, TIA_Manager and TIA_Admin; on every Service Action verify the role or ownership server-side.
```

---

## Seed the prompt template (manual)

Create the `ANALYZE_TICKET` v1 record in `PromptTemplate` using the System prompt, User prompt and JSON schema from section 5 of the design document (`Model` = your deployment name, `Temperature` = 0.2, `MaxOutputTokens` = 2500, `IsActive` = True).

## Tips for Mentor
- Give one step at a time; publish after each step so errors surface early.
- If Mentor skips a rule, send a follow-up such as "In Analysis_Execute add step 9 (urgency floor) which you left out."
- Check generated entities for correct identifier types and delete rules (children of Analysis must cascade).
- Always add secrets (API key, endpoint) by hand in the ODC Portal. Never put them in a prompt.
