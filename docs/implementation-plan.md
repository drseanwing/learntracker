# Clinical Education Impact Registry (CEIR)

## Comprehensive Implementation Plan

**Version:** 1.0
**Date:** 2026-02-05
**Author:** REdI Program, Metro North Health
**Status:** Architecture & Planning

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Decision Records](#2-architecture-decision-records)
3. [System Architecture](#3-system-architecture)
4. [Data Schema](#4-data-schema)
5. [Naming Conventions](#5-naming-conventions)
6. [Application Structure & Component Design](#6-application-structure--component-design)
7. [Flows & Logic](#7-flows--logic)
8. [Anonymous Survey System](#8-anonymous-survey-system)
9. [Report Generation](#9-report-generation)
10. [Access Control](#10-access-control)
11. [Error Handling & Resilience](#11-error-handling--resilience)
12. [Coding Standards](#12-coding-standards)
13. [Logging & Observability](#13-logging--observability)
14. [Testing Strategy](#14-testing-strategy)
15. [Deployment & ALM](#15-deployment--alm)
16. [Documentation Requirements](#16-documentation-requirements)
17. [Phased Delivery Plan](#17-phased-delivery-plan)
18. [Risk Register](#18-risk-register)
19. [Appendix A: Dataverse Table Definitions](#appendix-a-dataverse-table-definitions)
20. [Appendix B: Power Automate Flow Specifications](#appendix-b-power-automate-flow-specifications)
21. [Appendix C: UI Wireframe Specifications](#appendix-c-ui-wireframe-specifications)

---

## 1. Executive Summary

### 1.1 Purpose

The Clinical Education Impact Registry (CEIR) captures, tracks, and evaluates the impact of educational interventions — simulation events, in-service training, skills workshops — across their full lifecycle from planning through followup. It provides structured data collection with minimal free-text entry, anonymous participant feedback via QR-linked surveys, latent hazard and corrective action tracking, and automated professional reporting.

### 1.2 Key Capabilities

- **Intervention lifecycle management** — guided workflow through planning, execution, and followup phases with structured data capture (checkboxes, comboboxes, tags)
- **Anonymous participant feedback** — dynamically generated QR codes linking to lightweight HTML surveys, no authentication required
- **Issue & corrective action registry** — identification, tracking, and monitoring of latent hazards with escalation and resolution workflows
- **Automated reporting** — professional PDF/DOCX reports generated and emailed at intervention completion
- **Dashboard** — overview of open interventions, pending actions, and aggregate impact metrics

### 1.3 Technology Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Primary UI** | PowerApps Code Apps (React + TypeScript + Fluent UI v9) | Pro-code React SPA within Power Platform; Entra auth built-in; connector access from JS |
| **Data** | Dataverse | Relational, role-based security, full Power Automate integration, Premium licensed |
| **Automation** | Power Automate (cloud flows) | Report generation, email distribution, survey processing, reminders |
| **Anonymous surveys** | Power Automate HTTP trigger → HTML form → processing flow → Dataverse | Proven pattern, no extra licensing, anonymous access |
| **Reporting** | Power Automate + Word templates / HTML-to-PDF | Generates and emails professional reports |

### 1.4 Initial Scope

Solo educator (REdI team lead) as sole authenticated user. Architecture supports future multi-user and multi-team expansion but Phase 1 targets single-user workflow validation.

---

## 2. Architecture Decision Records

### ADR-001: PowerApps Code Apps as Primary UI

**Status:** Accepted (with mitigation plan)

**Context:** Code Apps is in Early Access Preview (as of Feb 2026). It allows full React SPAs to run within Power Platform with Entra auth, connector access, and organisational policy compliance. The user has existing access to the preview.

**Decision:** Use Code Apps as the primary UI platform.

**Rationale:**
- Enables full React component library with Fluent UI v9 for rich, responsive forms
- Provides Entra authentication out of the box — no auth code to write
- Direct JavaScript access to Power Platform connectors (Dataverse, Outlook, SharePoint)
- Hosted within Power Platform — organisational DLP and Conditional Access policies apply automatically
- Aligns with user's goal of experimenting with this technology
- TypeScript/React skillset is transferable if migration needed

**Risks & Mitigations:**
- **Preview instability**: Known Dataverse connector bugs. Mitigation: use the `@microsoft/power-apps` SDK's generated models; isolate data access in a service layer for easy swap to direct Dataverse Web API if needed.
- **No mobile app support**: Mitigation: responsive web design; the app runs in-browser on mobile devices.
- **No Git integration**: Mitigation: maintain a separate Git repository; use `pac code push` for deployment.
- **Future deprecation/change**: Mitigation: the service layer abstraction means the React UI components are portable to any React host (standalone SPA, Azure Static Web Apps, etc.) with data layer swap.

**Fallback:** If Code Apps proves unworkable during Phase 1, migrate to a standalone React SPA on the existing Azure VPS with Dataverse Web API (OAuth2 via MSAL). The React components, state management, and business logic remain identical — only the data access layer changes.

### ADR-002: Dataverse as Data Store

**Status:** Accepted

**Context:** Premium licensing is available. The application has relational data with multiple entity relationships, requires row-level security for future multi-user support, and must integrate tightly with Power Automate.

**Decision:** Use Dataverse as the sole persistent data store.

**Rationale:**
- Full relational model with lookups, many-to-many, and polymorphic relationships
- Built-in row-level security and business unit scoping for future access control
- Native Power Automate triggers (on create, update, delete)
- No delegation limits (unlike SharePoint connector)
- Audit logging built in
- Choice/OptionSet columns map directly to dropdown/checkbox UI patterns

**Trade-offs:**
- Dataverse has a learning curve for table/column configuration
- API request limits exist (but generous for single-user; ~40,000/day with Premium)
- Schema changes require solution management

### ADR-003: Power Automate HTTP Trigger for Anonymous Surveys

**Status:** Accepted

**Context:** Participants (including external clinicians, students, visitors) must provide feedback without M365 authentication. Power Pages would work but adds licensing complexity. Microsoft Forms lacks deep integration.

**Decision:** Use the existing proven pattern: Power Automate HTTP trigger serves an HTML form; form POST submits to a second processing flow which writes to Dataverse.

**Rationale:**
- Proven pattern already in use by the educator
- No additional licensing
- Full control over form design (can match Fluent UI styling)
- Query string parameters allow linking responses to specific sessions
- Anonymous — no Entra auth required for respondents

**Constraints:**
- HTTP trigger URLs change if the flow is recreated (not on save/update — only on delete+recreate)
- URLs are long and opaque — QR code is essential
- Need CORS headers if fetching via AJAX; simpler to use traditional form POST with redirect
- Rate limiting: Power Automate HTTP triggers are throttled at ~100 requests/minute (sufficient for survey use)

### ADR-004: Fluent UI v9 as Component Library

**Status:** Accepted

**Context:** Code Apps runs within Power Platform, which uses Fluent UI natively. Consistency with the host environment improves UX.

**Decision:** Use `@fluentui/react-components` (v9) as the primary UI library.

**Rationale:**
- Visual consistency with Power Platform host
- Accessible by default (WCAG 2.1 AA)
- Rich form components: Combobox, TagPicker, Checkbox, Dropdown, SpinButton
- Tree-shakeable — only imported components are bundled
- Actively maintained by Microsoft

### ADR-005: Report Generation via Power Automate

**Status:** Accepted

**Context:** Reports must be generated automatically and emailed. Options include client-side PDF generation, Power Automate Word templates, or custom API.

**Decision:** Use Power Automate with Word template population and PDF conversion.

**Rationale:**
- Word templates are editable by non-developers (future maintainability)
- Power Automate "Populate a Microsoft Word template" action handles merge fields
- "Convert file" action produces PDF
- "Send an email (V2)" distributes the report
- Stays entirely within M365 ecosystem — no external dependencies

---

## 3. System Architecture

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        POWER PLATFORM                               │
│                                                                     │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐  │
│  │   CODE APP (React)   │    │         DATAVERSE                 │  │
│  │                      │    │                                    │  │
│  │  ┌────────────────┐  │    │  Interventions    Sessions         │  │
│  │  │  UI Components │  │    │  Events           Attendance       │  │
│  │  │  (Fluent UI v9)│  │    │  People           Feedback         │  │
│  │  └───────┬────────┘  │    │  Issues           CorrectiveActions│  │
│  │          │           │    │  Equipment        Stakeholders     │  │
│  │  ┌───────▼────────┐  │    │  Evaluations      Tags            │  │
│  │  │ State Mgmt     │  │    │                                    │  │
│  │  │ (React Context)│  │    └──────────┬───────────────────────┘  │
│  │  └───────┬────────┘  │               │                          │
│  │          │           │               │                          │
│  │  ┌───────▼────────┐  │    ┌──────────▼───────────────────────┐  │
│  │  │ Service Layer  │◄─┼───►│    POWER APPS SDK               │  │
│  │  │ (Data Access)  │  │    │  (@microsoft/power-apps)         │  │
│  │  └────────────────┘  │    │  Generated models & services     │  │
│  │                      │    └──────────────────────────────────┘  │
│  └──────────────────────┘                                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    POWER AUTOMATE                             │  │
│  │                                                               │  │
│  │  ┌─────────────┐ ┌──────────────┐ ┌──────────────────────┐  │  │
│  │  │ Survey Serve │ │ Survey       │ │ Report Generator     │  │  │
│  │  │ (HTTP GET)   │ │ Process      │ │ (Word template +     │  │  │
│  │  │ → HTML form  │ │ (HTTP POST)  │ │  PDF convert + email)│  │  │
│  │  └─────────────┘ │ → Dataverse  │ └──────────────────────┘  │  │
│  │                   └──────────────┘                            │  │
│  │  ┌─────────────────┐ ┌───────────────────────────────────┐   │  │
│  │  │ Reminder/Nudge  │ │ Corrective Action Escalation      │   │  │
│  │  │ (Scheduled)     │ │ (Dataverse trigger)               │   │  │
│  │  └─────────────────┘ └───────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  ENTRA ID — Authentication & Authorization                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

         ┌──────────────────────────┐
         │  ANONYMOUS PARTICIPANTS  │
         │                          │
         │  Mobile/Desktop browser  │
         │  ──► QR code scan        │
         │  ──► HTML survey form    │
         │  ──► POST to PA flow     │
         └──────────────────────────┘
```

### 3.2 Data Flow Summary

1. **Educator creates Intervention** → Code App UI → Power Apps SDK → Dataverse
2. **Educator adds Events/Sessions** → Same path; Dataverse triggers generate survey URLs via Power Automate
3. **Session occurs** → Educator captures execution data in-app
4. **Participants scan QR** → HTTP GET flow serves HTML form → Participant submits → HTTP POST flow writes Feedback to Dataverse
5. **Issues identified** → Educator records in-app → Corrective Action created → Power Automate monitors due dates
6. **Intervention completed** → Educator triggers report → Power Automate generates Word/PDF → emails to stakeholders
7. **Dashboard** → Code App queries Dataverse for aggregate views

---

## 4. Data Schema

### 4.1 Entity Relationship Diagram (Conceptual)

```
Intervention (1)──────────(M) Event
     │                           │
     │                           │
     ├──(M) InterventionStakeholder    Event (1)──────(M) Session
     │                                        │
     ├──(M) InterventionEquipment              Session (1)──(M) SessionAttendance
     │                                                │            │
     ├──(M) InterventionBarrier                       │        Person (M)
     │                                                │
     ├──(M) InterventionTag                    Session (1)──(M) Feedback
     │                                                │
     └──(M) Evaluation                         Session (1)──(M) Issue
                                                              │
                                                       Issue (1)──(M) CorrectiveAction
```

### 4.2 Core Tables

#### Intervention
The top-level entity representing a strategy, program, or package of education.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_interventionid` | GUID (PK) | Primary key |
| `redi_name` | Text (200) | Intervention title |
| `redi_description` | Multiline text | Brief description |
| `redi_type` | Choice | Simulation, In-Service, Workshop, Skills Station, Orientation, Other |
| `redi_status` | Choice | Draft, Planning, Active, Completed, Cancelled |
| `redi_phase` | Choice | Planning, Execution, Followup, Closed |
| `redi_priority` | Choice | Routine, Urgent, Critical |
| `redi_targetunit` | Text (200) | Target ward/unit/department (e.g., "Cardiac Clinic") |
| `redi_targetfacility` | Text (200) | Facility name |
| `redi_learningneedsanalysis` | Choice | Formal LNA, Incident-driven, Request-driven, Gap analysis, Regulatory, Ad hoc |
| `redi_interventiondesign` | Choice (multi) | Didactic, Hands-on skills, Simulation (high-fidelity), Simulation (low-fidelity), Case-based discussion, Team training, Debriefing, Blended |
| `redi_endorsement` | Choice | Line manager, Nurse unit manager, Medical director, Education committee, Self-initiated, Other |
| `redi_plannedstartdate` | Date | Planned start |
| `redi_plannedenddate` | Date | Planned end |
| `redi_actualstartdate` | Date | Actual start |
| `redi_actualenddate` | Date | Actual end |
| `redi_planninghours` | Decimal | Total hours invested in planning |
| `redi_deliveryhours` | Decimal | Total hours of delivery |
| `redi_followuphours` | Decimal | Total hours invested in followup |
| `redi_overallvaluerating` | Choice | 1-5 Likert (Not valuable → Highly valuable) |
| `redi_overallvaluenotes` | Multiline text | Optional narrative on overall value |
| `redi_ownerid` | Lookup (User) | Owning educator |
| `redi_createdon` | DateTime | Auto |
| `redi_modifiedon` | DateTime | Auto |

#### Event
A specific activity or objective within an Intervention.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_eventid` | GUID (PK) | Primary key |
| `redi_interventionid` | Lookup (Intervention) | Parent intervention |
| `redi_name` | Text (200) | Event title |
| `redi_description` | Multiline text | Learning objectives / scenario description |
| `redi_eventtype` | Choice | Simulation scenario, Skills station, Lecture, Workshop, Assessment, Debrief, Other |
| `redi_status` | Choice | Planned, Scheduled, Completed, Cancelled |
| `redi_targetaudience` | Choice (multi) | Medical, Nursing, Allied health, Multidisciplinary, Students, Other |
| `redi_estimateddurationmins` | Integer | Planned duration in minutes |

#### Session
A specific delivery instance (same Event content, different time/group).

| Column | Type | Purpose |
|--------|------|---------|
| `redi_sessionid` | GUID (PK) | Primary key |
| `redi_eventid` | Lookup (Event) | Parent event |
| `redi_sessiondatetime` | DateTime | When the session occurred/will occur |
| `redi_location` | Text (200) | Room / simulation centre / ward |
| `redi_status` | Choice | Scheduled, In Progress, Completed, Cancelled |
| `redi_actualdurationmins` | Integer | Actual duration |
| `redi_participantcount` | Integer | Number of participants (auto-calculated or manual) |
| `redi_engagementrating` | Choice | 1-5 Likert |
| `redi_facilitatornotes` | Multiline text | Facilitator observations |
| `redi_surveyurl` | Text (500) | Auto-generated survey URL |
| `redi_surveyqrdata` | Text (2000) | QR code data URL (base64 PNG or URL) |
| `redi_surveyactive` | Boolean | Whether survey is accepting responses |

#### Person
Reusable person record (can be participant, facilitator, or stakeholder).

| Column | Type | Purpose |
|--------|------|---------|
| `redi_personid` | GUID (PK) | Primary key |
| `redi_displayname` | Text (200) | Full name |
| `redi_email` | Text (200) | Email (optional for anonymous) |
| `redi_role` | Choice | Consultant, Registrar, RMO, Nurse (RN), Nurse (EN), NUM, CNE, CNS, CNC, Allied Health, Student, Other |
| `redi_department` | Text (200) | Department/unit |
| `redi_facility` | Text (200) | Facility |
| `redi_isinternal` | Boolean | M365 account holder? |

#### SessionAttendance (Junction)
Links People to Sessions with their role in that session.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_sessionattendanceid` | GUID (PK) | Primary key |
| `redi_sessionid` | Lookup (Session) | Which session |
| `redi_personid` | Lookup (Person) | Who attended |
| `redi_attendancerole` | Choice | Participant, Facilitator, Observer, Confederate |
| `redi_attended` | Boolean | Actually attended (for no-shows) |

#### Feedback
Survey responses linked to a session. Supports both authenticated (in-app) and anonymous (QR survey) submissions.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_feedbackid` | GUID (PK) | Primary key |
| `redi_sessionid` | Lookup (Session) | Which session |
| `redi_personid` | Lookup (Person) | Respondent (nullable for anonymous) |
| `redi_feedbacktype` | Choice | Participant, Facilitator |
| `redi_isanonymous` | Boolean | Anonymous submission |
| `redi_overallrating` | Choice | 1-5 Likert |
| `redi_relevancerating` | Choice | 1-5 Likert (How relevant to your practice?) |
| `redi_confidencechange` | Choice | Much less confident, Less confident, No change, More confident, Much more confident |
| `redi_wouldrecommend` | Boolean | Would recommend to colleagues |
| `redi_freetextcomment` | Multiline text (500) | Optional open comment |
| `redi_submittedon` | DateTime | Timestamp |
| `redi_source` | Choice | In-app, QR Survey |

#### Issue
Latent hazards, safety concerns, or problems identified during sessions.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_issueid` | GUID (PK) | Primary key |
| `redi_sessionid` | Lookup (Session) | Session where identified |
| `redi_title` | Text (300) | Short description |
| `redi_description` | Multiline text | Detailed description |
| `redi_category` | Choice | Equipment, Environment, Knowledge gap, Process/protocol, Communication, Medication, Documentation, Infrastructure, Other |
| `redi_severity` | Choice | Low, Medium, High, Critical |
| `redi_status` | Choice | Identified, Under review, Action required, Resolved, Escalated, Accepted risk |
| `redi_identifiedby` | Lookup (Person) | Who identified |
| `redi_identifiedon` | DateTime | When identified |
| `redi_reportedto` | Text (200) | Who was notified |
| `redi_reportedvia` | Choice | Verbal, Email, Riskman/Incident report, Meeting, Other |
| `redi_riskmanreference` | Text (100) | External incident report reference |

#### CorrectiveAction
Actions taken to resolve identified issues.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_correctiveactionid` | GUID (PK) | Primary key |
| `redi_issueid` | Lookup (Issue) | Parent issue |
| `redi_title` | Text (300) | Action description |
| `redi_actiontype` | Choice | Immediate fix, Policy change, Training, Equipment replacement, Escalation, Environmental modification, Process redesign, Other |
| `redi_assignedto` | Text (200) | Responsible person/role |
| `redi_status` | Choice | Pending, In progress, Completed, Verified, Overdue |
| `redi_duedate` | Date | Target completion |
| `redi_completeddate` | Date | Actual completion |
| `redi_evidence` | Multiline text | Evidence of completion |
| `redi_verifiedby` | Text (200) | Who verified |

#### InterventionStakeholder (Junction)
People engaged during planning phase.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_interventionstakeholderid` | GUID (PK) | Primary key |
| `redi_interventionid` | Lookup (Intervention) | Which intervention |
| `redi_personid` | Lookup (Person) | Who |
| `redi_engagementtype` | Choice | Consulted, Endorsed, Co-designed, Informed, Requested, Approved |
| `redi_notes` | Text (500) | Context |

#### InterventionEquipment
Resources used/required.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_interventionequipmentid` | GUID (PK) | Primary key |
| `redi_interventionid` | Lookup (Intervention) | Which intervention |
| `redi_equipmentname` | Text (200) | Equipment item |
| `redi_category` | Choice | Manikin, Task trainer, AV equipment, Medication (simulated), Consumables, Documentation, IT/software, Furniture, Other |
| `redi_quantity` | Integer | How many |
| `redi_source` | Choice | REdI owned, Borrowed, Purchased, Ward stock, Simulated/printed |
| `redi_notes` | Text (500) | Condition, availability, issues |

#### InterventionBarrier
Barriers encountered during planning or delivery.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_interventionbarrierid` | GUID (PK) | Primary key |
| `redi_interventionid` | Lookup (Intervention) | Which intervention |
| `redi_barriertype` | Choice | Staffing/release, Space availability, Equipment availability, Time constraints, Stakeholder buy-in, Budget, Competing priorities, COVID/infection control, Other |
| `redi_impact` | Choice | Minor delay, Significant delay, Scope reduction, Cancellation, Workaround found |
| `redi_notes` | Text (500) | Details |

#### InterventionTag (Junction to Tag)
Flexible tagging for categorisation and filtering.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_interventiontagid` | GUID (PK) | Primary key |
| `redi_interventionid` | Lookup (Intervention) | Which intervention |
| `redi_tagid` | Lookup (Tag) | Which tag |

#### Tag
Reusable tags for flexible categorisation.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_tagid` | GUID (PK) | Primary key |
| `redi_name` | Text (100) | Tag label (e.g., "ALS", "Cardiac arrest", "Medication safety") |
| `redi_category` | Choice | Topic, Skill, Domain, Clinical area, Framework, Custom |

#### Evaluation
Followup evaluation data for the intervention.

| Column | Type | Purpose |
|--------|------|---------|
| `redi_evaluationid` | GUID (PK) | Primary key |
| `redi_interventionid` | Lookup (Intervention) | Which intervention |
| `redi_kirkpatricklevel` | Choice | L1 Reaction, L2 Learning, L3 Behaviour, L4 Results |
| `redi_method` | Choice | Survey, Observation, Audit, Interview, Chart review, Pre-post test, Simulation assessment, Self-report, Other |
| `redi_findings` | Multiline text | What was found |
| `redi_evidenceofimpact` | Choice | No evidence, Anecdotal, Moderate evidence, Strong evidence |
| `redi_evaluationdate` | Date | When evaluation occurred |
| `redi_evaluator` | Text (200) | Who performed the evaluation |

#### ChangeImplemented
Changes made as a result of the intervention (followup phase).

| Column | Type | Purpose |
|--------|------|---------|
| `redi_changeimplementedid` | GUID (PK) | Primary key |
| `redi_interventionid` | Lookup (Intervention) | Which intervention |
| `redi_changetype` | Choice | Protocol/guideline update, Equipment change, Environmental modification, Staffing change, Training program, Documentation update, Escalation pathway, Other |
| `redi_description` | Multiline text | What changed |
| `redi_implementeddate` | Date | When implemented |
| `redi_status` | Choice | Proposed, Implemented, Monitoring, Sustained, Reverted |
| `redi_monitoringmethod` | Text (500) | How sustainability is being monitored |

### 4.3 Choice Column Definitions

All Choice (OptionSet) columns are defined as **local** choices (scoped to the table) unless reused across 3+ tables, in which case they become **global** choices.

**Global Choices:**
- `redi_LikertScale5`: 1 (Strongly disagree/Not at all) through 5 (Strongly agree/Extremely)
- `redi_StatusGeneric`: Draft, Active, Completed, Cancelled
- `redi_SeverityLevel`: Low, Medium, High, Critical

All other choices remain local to their table.

### 4.4 Calculated/Rollup Columns

| Table | Column | Type | Logic |
|-------|--------|------|-------|
| Intervention | `redi_totalsessions` | Rollup | COUNT of Sessions across all Events |
| Intervention | `redi_totalparticipants` | Rollup | COUNT of distinct SessionAttendance (role=Participant) |
| Intervention | `redi_avgfeedbackrating` | Rollup | AVG of Feedback.overallrating across all Sessions |
| Intervention | `redi_openissuecount` | Rollup | COUNT of Issues where status != Resolved |
| Intervention | `redi_openactioncount` | Rollup | COUNT of CorrectiveActions where status != Completed/Verified |
| Session | `redi_feedbackcount` | Rollup | COUNT of related Feedback |
| Session | `redi_avgrating` | Rollup | AVG of related Feedback.overallrating |
| Issue | `redi_openactioncount` | Rollup | COUNT of CorrectiveActions where status != Completed |

---

## 5. Naming Conventions

### 5.1 Dataverse

| Element | Convention | Example |
|---------|-----------|---------|
| Publisher prefix | `redi` | `redi_` |
| Solution name | PascalCase | `ClinicalEducationImpactRegistry` |
| Table logical name | lowercase, no spaces | `redi_intervention` |
| Table display name | PascalCase singular | `Intervention` |
| Column logical name | lowercase, no spaces | `redi_targetunit` |
| Column display name | Title Case | `Target Unit` |
| Choice (OptionSet) | PascalCase with prefix | `redi_InterventionType` |
| Relationship name | ParentTable_ChildTable | `redi_Intervention_Event` |

### 5.2 Code App (React/TypeScript)

| Element | Convention | Example |
|---------|-----------|---------|
| Component files | PascalCase.tsx | `InterventionForm.tsx` |
| Hook files | camelCase.ts | `useIntervention.ts` |
| Service files | camelCase.ts | `interventionService.ts` |
| Type/Interface files | PascalCase.types.ts | `Intervention.types.ts` |
| Constants | UPPER_SNAKE_CASE | `MAX_FREE_TEXT_LENGTH` |
| Functions | camelCase | `getOpenInterventions()` |
| React components | PascalCase | `<SessionCard />` |
| CSS modules | camelCase.module.css | `interventionForm.module.css` |
| State variables | camelCase | `interventionStatus` |
| Boolean variables | is/has/should prefix | `isLoading`, `hasUnsavedChanges` |

### 5.3 Power Automate

| Element | Convention | Example |
|---------|-----------|---------|
| Flow name | CEIR - [Trigger] - [Action] | `CEIR - Session Created - Generate Survey URL` |
| Flow folder | CEIR | All flows grouped |
| Variable names | camelCase | `sessionId`, `surveyHtml` |
| Compose actions | desc_PurposeName | `desc_BuildSurveyUrl` |
| Scope names | scope_PhaseName | `scope_ValidateInput` |
| Error handling scopes | scope_TryCatch_ActionName | `scope_TryCatch_SendEmail` |

---

## 6. Application Structure & Component Design

### 6.1 Project Structure

```
ceir-app/
├── src/
│   ├── index.tsx                      # App entry point
│   ├── App.tsx                        # Root component, routing
│   ├── components/
│   │   ├── layout/
│   │   │   ├── AppShell.tsx           # Navigation, header, sidebar
│   │   │   ├── NavMenu.tsx            # Primary navigation
│   │   │   └── PageContainer.tsx      # Content wrapper
│   │   ├── dashboard/
│   │   │   ├── DashboardPage.tsx      # Main dashboard view
│   │   │   ├── OpenInterventionsList.tsx
│   │   │   ├── PendingActionsBanner.tsx
│   │   │   └── ImpactSummaryCards.tsx
│   │   ├── intervention/
│   │   │   ├── InterventionListPage.tsx
│   │   │   ├── InterventionDetailPage.tsx
│   │   │   ├── InterventionWizard.tsx  # Guided creation flow
│   │   │   ├── PlanningPhaseForm.tsx
│   │   │   ├── ExecutionPhaseForm.tsx
│   │   │   ├── FollowupPhaseForm.tsx
│   │   │   ├── StakeholderManager.tsx
│   │   │   ├── EquipmentManager.tsx
│   │   │   ├── BarrierManager.tsx
│   │   │   └── TagSelector.tsx
│   │   ├── event/
│   │   │   ├── EventListPanel.tsx
│   │   │   ├── EventForm.tsx
│   │   │   └── EventCard.tsx
│   │   ├── session/
│   │   │   ├── SessionListPanel.tsx
│   │   │   ├── SessionForm.tsx
│   │   │   ├── SessionCard.tsx
│   │   │   ├── AttendanceManager.tsx
│   │   │   └── QRCodeDisplay.tsx
│   │   ├── feedback/
│   │   │   ├── FeedbackSummary.tsx
│   │   │   ├── FeedbackDetailList.tsx
│   │   │   └── FeedbackCharts.tsx
│   │   ├── issues/
│   │   │   ├── IssueListPanel.tsx
│   │   │   ├── IssueForm.tsx
│   │   │   ├── IssueCard.tsx
│   │   │   ├── CorrectiveActionForm.tsx
│   │   │   └── CorrectiveActionTracker.tsx
│   │   ├── evaluation/
│   │   │   ├── EvaluationForm.tsx
│   │   │   ├── ChangeImplementedForm.tsx
│   │   │   └── ValueAssessmentForm.tsx
│   │   ├── reporting/
│   │   │   ├── ReportPreview.tsx
│   │   │   └── ReportTriggerButton.tsx
│   │   └── shared/
│   │       ├── PersonPicker.tsx        # Reusable person search/create
│   │       ├── LikertScale.tsx         # 1-5 rating component
│   │       ├── StatusBadge.tsx
│   │       ├── ConfirmDialog.tsx
│   │       ├── ErrorBoundary.tsx
│   │       ├── LoadingSpinner.tsx
│   │       └── EmptyState.tsx
│   ├── services/
│   │   ├── dataverse/
│   │   │   ├── interventionService.ts
│   │   │   ├── eventService.ts
│   │   │   ├── sessionService.ts
│   │   │   ├── feedbackService.ts
│   │   │   ├── issueService.ts
│   │   │   ├── personService.ts
│   │   │   └── tagService.ts
│   │   ├── reporting/
│   │   │   └── reportService.ts       # Triggers PA flow for reports
│   │   └── logger.ts                  # Application logging service
│   ├── hooks/
│   │   ├── useIntervention.ts
│   │   ├── useSession.ts
│   │   ├── useDataverseQuery.ts       # Generic query hook with caching
│   │   ├── useAutoSave.ts
│   │   └── useNavigationGuard.ts      # Unsaved changes protection
│   ├── types/
│   │   ├── intervention.types.ts
│   │   ├── event.types.ts
│   │   ├── session.types.ts
│   │   ├── feedback.types.ts
│   │   ├── issue.types.ts
│   │   ├── person.types.ts
│   │   └── common.types.ts
│   ├── constants/
│   │   ├── choiceOptions.ts           # All dropdown options centralised
│   │   ├── routes.ts
│   │   └── config.ts
│   ├── utils/
│   │   ├── dateUtils.ts
│   │   ├── validationUtils.ts
│   │   ├── qrCodeUtils.ts
│   │   └── formatUtils.ts
│   └── styles/
│       ├── theme.ts                   # Fluent UI v9 custom theme tokens
│       └── global.css
├── power.config.json                  # Power Apps SDK config (auto-generated)
├── package.json
├── tsconfig.json
└── .eslintrc.json
```

### 6.2 Key UI Patterns

#### Guided Wizard (Intervention Creation)
The `InterventionWizard` component implements a stepped form:

1. **Basics** → Name, type, target unit, priority, description
2. **Learning Needs** → Analysis type, endorsement, target audience (all Choice dropdowns)
3. **Design** → Intervention design methods (multi-select checkboxes), planned dates
4. **Stakeholders** → Add people engaged (PersonPicker + engagement type dropdown)
5. **Equipment** → Add resources (combobox + category dropdown)
6. **Barriers** → Record known barriers (type dropdown + impact dropdown)
7. **Tags** → Select/create tags (TagPicker)
8. **Review** → Summary card before save

Each step validates before advancing. Progress is auto-saved to Dataverse on step completion (draft status).

#### Minimal Free-Text Principle
All forms prioritise structured input:
- **Choice/Dropdown** for single-select categorical data
- **Checkbox groups** for multi-select options
- **ComboBox** for searchable lists with type-ahead (e.g., person names, equipment)
- **TagPicker** for flexible tagging
- **Likert scales** (custom `<LikertScale>` component) for ratings
- **SpinButton** for numeric values (duration, counts)
- **Free text** reserved only for: descriptions, notes, evidence fields — always optional, always with character limit guidance

#### Phase-Based Navigation
The `InterventionDetailPage` uses a horizontal tab bar reflecting the intervention lifecycle:

```
[ Planning ] [ Events & Sessions ] [ Issues & Actions ] [ Followup ] [ Report ]
```

The current phase is highlighted; future phases are accessible but show completion prompts.

### 6.3 State Management

React Context + useReducer for intervention-level state (avoids Redux overhead for a single-user app). Each major domain has its own context provider:

- `InterventionContext` — current intervention, events, sessions
- `AppContext` — user info, navigation state, global settings

Data fetching uses a custom `useDataverseQuery` hook wrapping the Power Apps SDK, with:
- Automatic loading/error state management
- Response caching (stale-while-revalidate pattern)
- Retry logic (3 attempts with exponential backoff)

---

## 7. Flows & Logic

### 7.1 Intervention Lifecycle State Machine

```
                    ┌──────────┐
                    │  Draft   │
                    └────┬─────┘
                         │ save basics
                         ▼
                    ┌──────────┐
              ┌─────│ Planning │
              │     └────┬─────┘
              │          │ mark planning complete
   cancel     │          ▼
   at any ────┤    ┌──────────┐
   point      │    │  Active  │ (execution phase)
              │    └────┬─────┘
              │         │ all sessions completed
              │         ▼
              │    ┌──────────┐
              │    │ Followup │
              │    └────┬─────┘
              │         │ generate report
              │         ▼
              │    ┌──────────┐
              └───►│ Closed   │
                   └──────────┘
                        │
                   ┌────▼─────┐
                   │Cancelled │
                   └──────────┘
```

**Transition Rules:**
- Draft → Planning: Requires name, type, target unit
- Planning → Active: Requires at least 1 event with at least 1 session scheduled
- Active → Followup: All sessions completed OR manually advanced
- Followup → Closed: Report generated (can reopen to Followup)
- Any → Cancelled: With reason (stored in notes)

### 7.2 Session Survey Lifecycle

1. Session created → Status: Scheduled
2. Session marked "In Progress" or "Completed" → Power Automate trigger generates survey URL
3. Survey URL + QR code stored on Session record
4. Survey responses flow in via anonymous HTTP flow → written as Feedback rows
5. Educator deactivates survey (sets `surveyactive = false`) → HTTP flow rejects submissions with "Survey closed" message

### 7.3 Corrective Action Monitoring

Power Automate scheduled flow (daily at 08:00):
1. Query CorrectiveActions where `status = Pending or In Progress` AND `duedate <= today + 3 days`
2. For items due within 3 days: send reminder email to intervention owner
3. For overdue items: update status to "Overdue"; send escalation email
4. Log all notifications to a simple Dataverse audit log

### 7.4 Report Generation Trigger

1. Educator clicks "Generate Report" in the Followup tab
2. Code App calls a Power Automate HTTP trigger (authenticated via service connection)
3. Flow queries all intervention data, child records, aggregates
4. Populates Word template with merge fields
5. Converts to PDF
6. Emails to educator + any specified stakeholders
7. Stores PDF link back on Intervention record

---

## 8. Anonymous Survey System

### 8.1 Architecture

```
┌─────────────┐     GET /survey?sid={sessionId}&token={hmac}
│  QR Code     │────────────────────────────────────────────►┌─────────────────┐
│  (on screen  │                                              │ PA Flow: Survey │
│  or printed) │                                              │ Serve (HTTP GET)│
└─────────────┘                                              └────────┬────────┘
                                                                      │
                                                              Validates token
                                                              Checks surveyactive
                                                              Returns HTML form
                                                                      │
                                                                      ▼
┌─────────────┐     POST form data                           ┌─────────────────┐
│  Participant │────────────────────────────────────────────►│ PA Flow: Survey │
│  Browser     │                                              │ Process (HTTP)  │
└─────────────┘                                              └────────┬────────┘
                                                                      │
                                                              Validates input
                                                              Creates Feedback row
                                                              Returns thank-you HTML
                                                                      │
                                                                      ▼
                                                              ┌─────────────────┐
                                                              │    DATAVERSE    │
                                                              │  Feedback table  │
                                                              └─────────────────┘
```

### 8.2 Survey URL Structure

```
https://{flow-endpoint}/survey?sid={sessionGuid}&t={hmacToken}
```

- `sid`: Session GUID — identifies which session the feedback belongs to
- `t`: HMAC-SHA256 token derived from `sid + secret` — prevents URL tampering / enumeration
- The secret is stored as an Environment Variable in Dataverse (not hardcoded in flows)

### 8.3 QR Code Generation

The Code App generates QR codes client-side using a lightweight library (`qrcode` npm package rendered to canvas/data URL). The QR image is displayed in-app and can be:
- Shown full-screen for scanning during a session
- Downloaded as PNG for printing
- The data URL is stored on the Session record for the report

### 8.4 Survey Form Design

The HTML form served by the GET flow is self-contained (inline CSS, no external dependencies) and designed for mobile-first:

**Questions (default set — configurable per Event in future):**

1. **Overall rating** — 5-star visual scale (radio buttons styled as stars)
2. **Relevance to practice** — 5-point Likert (radio group)
3. **Confidence change** — 5-point scale from "Much less" to "Much more" (radio group)
4. **Would recommend** — Yes/No toggle
5. **Open comment** — Optional textarea (max 500 chars)

Hidden fields: `sessionId`, `token`, `source=qr_survey`

The form POST submits to the processing flow URL (also encoded in a hidden field). On success, a "Thank you" confirmation page is returned.

### 8.5 Security Considerations

- HMAC token prevents arbitrary session ID guessing
- No PII collected (anonymous by default; optional name/role fields could be added)
- Rate limiting inherent in Power Automate HTTP triggers (~100 req/min)
- Survey can be deactivated by educator (flow checks `surveyactive` flag)
- Form includes basic input validation (required fields, max lengths)
- No CORS needed — traditional form POST with full-page redirect

---

## 9. Report Generation

### 9.1 Report Content Structure

The auto-generated report follows this structure:

1. **Header** — REdI branding, report date, intervention title, status
2. **Executive Summary** — Auto-generated from structured fields (type, target, dates, total participants, overall rating)
3. **Planning Phase**
   - Learning needs analysis type
   - Intervention design methods
   - Stakeholders engaged (table)
   - Equipment used (table)
   - Barriers encountered (table)
   - Total planning hours
4. **Delivery Phase**
   - Events and sessions summary (table: date, location, participants, duration)
   - Aggregate engagement metrics
   - Facilitator observations (collated)
5. **Participant Feedback**
   - Response rate
   - Rating distributions (overall, relevance, confidence change)
   - Recommendation rate
   - Selected comments (anonymised)
6. **Issues & Corrective Actions**
   - Issues identified (table: title, category, severity, status)
   - Corrective actions (table: action, type, status, due/completed date)
   - Outstanding actions highlighted
7. **Followup & Evaluation**
   - Evaluation methods and findings
   - Kirkpatrick level achieved
   - Evidence of impact
   - Changes implemented
8. **Overall Assessment**
   - Value rating with narrative
   - Total resource investment (planning + delivery + followup hours)
   - Recommendations for future iterations

### 9.2 Word Template Design

A Word template (`.docx`) stored in a SharePoint document library, with Content Controls / merge fields mapped to flow variables. The template uses:
- Repeating section content controls for tables (stakeholders, equipment, sessions, issues)
- Plain text content controls for single values
- Rich text controls for narrative sections
- Conditional sections (hidden if no data — handled in flow logic by populating with "N/A" or removing section)

### 9.3 Distribution

The generated PDF is:
1. Saved to a SharePoint document library (`CEIR Reports/{InterventionName}/`)
2. Emailed to the intervention owner
3. Optionally emailed to stakeholders flagged as "Informed" in the stakeholder list
4. A link to the SharePoint file is stored on the Intervention Dataverse record

---

## 10. Access Control

### 10.1 Role Model

Three roles, implemented as Dataverse Security Roles:

| Role | Permissions | Phase 1 Scope |
|------|------------|---------------|
| **Educator** | Full CRUD on own interventions and all child records; read-only on other educators' completed interventions; can trigger own reports | Primary user role (Sean) |
| **Manager** | All Educator permissions + read/write all interventions in their business unit; can reassign interventions; receives escalation emails | Future: team lead / NUM |
| **Admin** | Full system access; manage Choice lists, tags, Word templates; manage security roles; view audit logs | Future: system administrator |

### 10.2 Implementation

**Phase 1 (solo user):** Single Educator security role with owner-scoped permissions. Dataverse row ownership provides implicit access control.

**Phase 2+ (multi-user):** Business Unit hierarchy maps to facility/department structure. Security roles scoped to Business Unit for Manager role. Column-level security on sensitive fields (e.g., Riskman references) if needed.

### 10.3 Anonymous Survey Access

The survey flows operate under a service account connection (not user-delegated). The HTTP trigger is unauthenticated by design. Security is provided by:
- HMAC token validation (prevents URL guessing)
- Survey active/inactive flag (time-bounded access)
- No read access to existing data via the survey endpoint
- Write-only to Feedback table

---

## 11. Error Handling & Resilience

### 11.1 Code App (React)

**Error Boundary hierarchy:**
```
<AppErrorBoundary>              ← Catches fatal errors, shows recovery UI
  <AppShell>
    <PageErrorBoundary>         ← Per-page boundary, allows navigation away
      <InterventionDetailPage>
        <SectionErrorBoundary>  ← Per-section, other sections remain usable
          <FeedbackSummary />
        </SectionErrorBoundary>
      </InterventionDetailPage>
    </PageErrorBoundary>
  </AppShell>
</AppErrorBoundary>
```

**Service layer error handling pattern:**
```typescript
// Every service method follows this pattern
async function getIntervention(id: string): Promise<Result<Intervention>> {
  try {
    logger.debug('interventionService.get', { id });
    const response = await dataverseClient.get('redi_interventions', id);
    logger.info('interventionService.get.success', { id });
    return { success: true, data: mapToIntervention(response) };
  } catch (error) {
    logger.error('interventionService.get.failed', { id, error });
    if (isNetworkError(error)) {
      return { success: false, error: 'NETWORK_ERROR', message: 'Unable to connect. Check your connection.' };
    }
    if (isNotFoundError(error)) {
      return { success: false, error: 'NOT_FOUND', message: 'Intervention not found.' };
    }
    return { success: false, error: 'UNKNOWN', message: 'An unexpected error occurred.' };
  }
}
```

**Graceful degradation:**
- Network failure → cached data displayed with "offline" indicator; writes queued
- Partial data load failure → affected section shows error state; rest of page functional
- Survey generation failure → manual URL display with copy button as fallback

### 11.2 Power Automate Flows

**Every flow uses the Scope-based try/catch pattern:**

```
Scope: Try
  ├── [Business logic actions]
  └── [Set success variables]

Scope: Catch (Configure Run After: Try has failed/timed out)
  ├── Compose: Error details (from outputs of failed actions)
  ├── Create row: ErrorLog table in Dataverse
  ├── Send email: Notification to admin/owner
  └── [Return appropriate HTTP response if HTTP-triggered]

Scope: Finally (Configure Run After: Catch has succeeded/failed/skipped)
  └── [Cleanup actions]
```

**HTTP-triggered flows additionally return:**
- 200 + HTML on success
- 400 + error HTML on validation failure
- 404 + error HTML if session not found or survey inactive
- 500 + generic error HTML on unexpected failure

### 11.3 Retry Policy

| Scenario | Retry Count | Backoff | Notes |
|----------|-------------|---------|-------|
| Dataverse API call (Code App) | 3 | Exponential (1s, 2s, 4s) | On 429/5xx only |
| Power Automate Dataverse action | Default PA retry | Exponential | Built-in |
| Email sending | 2 | Fixed 30s | On transient failure |
| Survey form submission | 0 | N/A | User can resubmit manually |

---

## 12. Coding Standards

### 12.1 TypeScript

- **Strict mode enabled** (`"strict": true` in tsconfig)
- **No `any` type** — use `unknown` + type guards where type is uncertain
- **Explicit return types** on all exported functions
- **Interface over type** for object shapes (extensibility)
- **Enums** for finite sets that map to Dataverse Choice values
- **Barrel exports** (`index.ts`) for each feature folder

### 12.2 React

- **Functional components only** (no class components)
- **Custom hooks** for all stateful logic and side effects
- **Props interface** defined above each component, exported from component file
- **Memoisation** (`useMemo`, `useCallback`) for expensive computations and stable callback references
- **Key prop** always derived from record ID (never array index)
- **Controlled components** for all form inputs
- **Single responsibility** — components do one thing; compose for complexity

### 12.3 Comments & Documentation

```typescript
/**
 * Retrieves all sessions for a given event, including attendance counts.
 *
 * @param eventId - The Dataverse GUID of the parent Event
 * @returns Array of Session objects with computed attendanceCount
 * @throws {NetworkError} If Dataverse is unreachable
 *
 * @example
 * const sessions = await getSessionsForEvent('abc-123-def');
 */
export async function getSessionsForEvent(eventId: string): Promise<Session[]> {
  // Expand attendance to get count without separate query
  // Dataverse $expand with $count is more efficient than separate call
  ...
}
```

- **Every exported function** has JSDoc with `@param`, `@returns`, and `@throws`
- **Every component** has a brief JSDoc describing its purpose and key props
- **Inline comments** for non-obvious logic (the "why", not the "what")
- **TODO/FIXME** comments include ticket/issue reference

### 12.4 Modularisation Rules

- No file exceeds 300 lines (refactor into sub-components or utility functions)
- No component accepts more than 8 props (create a config object or split)
- Service layer is the only code that imports Power Apps SDK — components never call Dataverse directly
- All Dataverse column names are abstracted behind TypeScript interfaces (components never reference `redi_` prefixed names)

---

## 13. Logging & Observability

### 13.1 Application Logging

A centralised `logger.ts` service with severity levels:

```typescript
enum LogLevel { DEBUG, INFO, WARN, ERROR }

interface LogEntry {
  timestamp: string;
  level: LogLevel;
  context: string;        // e.g., 'interventionService.create'
  message: string;
  data?: Record<string, unknown>;
  userId?: string;
  sessionId?: string;
}
```

**Log destinations (Phase 1):**
- Browser console (all levels in development; WARN+ in production)
- In-memory circular buffer (last 500 entries) — accessible via debug panel
- On ERROR: POST to a Power Automate HTTP flow that writes to a Dataverse `redi_applog` table

**Future (Phase 2+):**
- Azure Application Insights integration (when Code Apps adds native support, or via manual SDK)

### 13.2 What to Log

| Level | When | Examples |
|-------|------|---------|
| DEBUG | Detailed flow tracing (dev only) | Function entry/exit, intermediate state |
| INFO | Significant business events | Intervention created, session completed, report generated |
| WARN | Recoverable issues | Retry triggered, slow response (>3s), data validation warning |
| ERROR | Failures requiring attention | API call failed after retries, unhandled exception, flow error |

### 13.3 Power Automate Logging

Every flow writes key actions to a shared `redi_flowlog` Dataverse table:

| Column | Purpose |
|--------|---------|
| `redi_flowname` | Which flow |
| `redi_runid` | PA run ID for correlation |
| `redi_action` | What was attempted |
| `redi_status` | Success / Failure |
| `redi_details` | Error message or summary |
| `redi_timestamp` | When |

---

## 14. Testing Strategy

### 14.1 Code App Testing

| Layer | Tool | Coverage Target |
|-------|------|----------------|
| Unit (utilities, mappers) | Vitest | 90%+ |
| Component (render, interaction) | React Testing Library | Key interaction paths |
| Integration (service layer) | Vitest + MSW (mock service worker) | All CRUD operations |
| E2E | Manual testing (Phase 1) | Critical user journeys |

**Phase 1 pragmatic approach:** Given solo developer context, prioritise:
1. Service layer unit tests (data mapping, validation logic)
2. Utility function tests (date formatting, HMAC generation, QR encoding)
3. Manual testing checklist for each user journey

### 14.2 Power Automate Testing

- Test each flow in isolation using the PA test runner
- For HTTP-triggered flows: test with Postman/curl collection
- Maintain a test Session record in Dataverse for survey flow testing
- Validate report output against expected template rendering

### 14.3 Data Validation

Validation occurs at two layers:

1. **UI layer (React)** — immediate user feedback; prevents submission of obviously invalid data
2. **Dataverse layer** — business rules and column constraints as the authoritative validation

| Rule | UI | Dataverse |
|------|----|---------| 
| Required fields | Form validation | Column: Business Required |
| Text max length | Input maxLength prop | Column: Max Length |
| Choice values | Dropdown restricts options | OptionSet enforces valid values |
| Date logic (end ≥ start) | Form validation | Business Rule |
| Referential integrity | Service layer check | Lookup relationship |

---

## 15. Deployment & ALM

### 15.1 Solution Strategy

**Dataverse Solution:** `ClinicalEducationImpactRegistry` (managed solution for production, unmanaged for development)

Contains:
- All custom tables, columns, relationships
- All Choice (OptionSet) definitions
- Security roles
- Environment variables (survey HMAC secret, report template location)
- Power Automate cloud flows (embedded in solution)

**Code App:** Deployed separately via `pac code push` (Code Apps don't yet support solution packaging)

### 15.2 Environment Strategy

| Environment | Purpose | Solution Type |
|-------------|---------|--------------|
| Dev | Development and testing | Unmanaged |
| Production | Live use | Managed (future) |

**Phase 1:** Single environment (Dev = Production). This is acceptable for a solo user. Formal Dev/Prod split introduced when additional users onboard.

### 15.3 Version Control

- **Code App source:** Git repository (GitHub or Azure DevOps)
- **Dataverse solution:** Exported as unmanaged solution ZIP and committed to Git after each schema change
- **Power Automate flows:** Included in Dataverse solution export
- **Word templates:** Version-controlled in Git; deployed to SharePoint

### 15.4 Deployment Checklist

```
[ ] Dataverse solution exported and committed to Git
[ ] All flows tested in target environment
[ ] Environment variables set (HMAC secret, template location, email addresses)
[ ] Security roles assigned
[ ] Word report template uploaded to SharePoint
[ ] Code App pushed via `pac code push`
[ ] Smoke test: create intervention → add event → add session → survey URL generates → submit feedback → generate report
```

---

## 16. Documentation Requirements

### 16.1 Maintained Documents

| Document | Location | Update Trigger |
|----------|----------|---------------|
| This Implementation Plan | Git repo root | Architecture changes |
| Data Dictionary | Git repo `/docs/data-dictionary.md` | Any schema change |
| Flow Specifications | Git repo `/docs/flows/` | Any flow change |
| User Guide | Git repo `/docs/user-guide.md` | Feature release |
| Runbook (operational) | Git repo `/docs/runbook.md` | Deployment or incident |
| Decision Log | Git repo `/docs/decisions.md` | Any ADR added |

### 16.2 Code Documentation

- README.md in repo root: setup instructions, architecture overview, deployment steps
- Each service file: JSDoc header explaining the domain and key patterns
- Complex components: brief comment block explaining state management approach
- Inline `// HACK:` or `// WORKAROUND:` with explanation for any Code Apps preview workarounds

---

## 17. Phased Delivery Plan

### Phase 1: Foundation (Weeks 1-4)

**Goal:** Core data model + basic intervention CRUD + session management

- [ ] Dataverse solution: create all tables, columns, relationships, choices
- [ ] Code App scaffold: project setup, routing, AppShell, theme
- [ ] Service layer: interventionService, eventService, sessionService
- [ ] InterventionWizard (basics + design steps only)
- [ ] InterventionListPage + InterventionDetailPage (planning tab)
- [ ] EventForm + SessionForm (basic CRUD)
- [ ] PersonPicker (shared component)
- [ ] Error boundaries + logging service
- [ ] Git repository setup

**Milestone:** Can create an intervention, add events and sessions, view list

### Phase 2: Survey System (Weeks 5-7)

**Goal:** Anonymous feedback collection via QR code

- [ ] Power Automate: Survey Serve flow (HTTP GET → HTML form)
- [ ] Power Automate: Survey Process flow (HTTP POST → Dataverse Feedback)
- [ ] HMAC token generation utility
- [ ] QRCodeDisplay component
- [ ] Session detail: survey URL display + QR + active toggle
- [ ] FeedbackSummary component (ratings, charts)
- [ ] Power Automate: Session Created → Generate Survey URL flow

**Milestone:** Can generate QR code, scan to fill survey, see responses in app

### Phase 3: Issues & Actions (Weeks 8-9)

**Goal:** Latent hazard identification and corrective action tracking

- [ ] IssueForm + IssueCard + IssueListPanel
- [ ] CorrectiveActionForm + CorrectiveActionTracker
- [ ] Power Automate: Daily corrective action reminder flow
- [ ] Issues tab in InterventionDetailPage
- [ ] StatusBadge for issue/action status visualisation

**Milestone:** Can record issues, create corrective actions, receive reminders

### Phase 4: Followup & Reporting (Weeks 10-13)

**Goal:** Evaluation capture + automated report generation

- [ ] EvaluationForm, ChangeImplementedForm, ValueAssessmentForm
- [ ] Followup tab in InterventionDetailPage
- [ ] Word report template design
- [ ] Power Automate: Report generation flow (query → populate template → PDF → email)
- [ ] ReportTriggerButton + ReportPreview component
- [ ] Planning phase forms completion (StakeholderManager, EquipmentManager, BarrierManager)

**Milestone:** Full intervention lifecycle from planning to generated PDF report

### Phase 5: Dashboard & Polish (Weeks 14-16)

**Goal:** Dashboard overview + UX refinement

- [ ] DashboardPage: open interventions, pending actions, impact metrics
- [ ] ImpactSummaryCards (aggregate stats)
- [ ] FeedbackCharts (distribution visualisations)
- [ ] Navigation guard (unsaved changes prompt)
- [ ] Auto-save implementation
- [ ] Accessibility audit and fixes
- [ ] User guide documentation

**Milestone:** Production-ready for solo educator use

---

## 18. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| R1 | Code Apps preview breaking change | Medium | High | Service layer abstraction; fallback to standalone React SPA documented in ADR-001 |
| R2 | Dataverse connector bugs in Code Apps | Medium | Medium | Isolate data access; can switch to direct Web API calls if SDK fails |
| R3 | Power Automate HTTP trigger URL changes | Low | Medium | URL stored in Environment Variable; monitoring flow checks URL validity weekly |
| R4 | Survey spam/abuse | Low | Low | HMAC token validation; survey deactivation; PA rate limiting |
| R5 | Report template rendering issues | Medium | Low | Test template with all field combinations; fallback to HTML email report |
| R6 | Scope creep (multi-user features before validation) | Medium | Medium | Phase 1 explicitly single-user; defer access control implementation |
| R7 | Dataverse API throttling | Low | Low | Single user generates minimal load; implement request batching if needed |
| R8 | Loss of Code Apps early access | Low | High | Same mitigation as R1; React code is portable |

---

## Appendix A: Dataverse Table Definitions

*(Detailed column specifications are in Section 4.2. This appendix captures additional configuration.)*

### Table Configuration Settings

| Table | Ownership | Change Tracking | Audit | Notes |
|-------|-----------|----------------|-------|-------|
| Intervention | User | Yes | Yes | Primary entity |
| Event | Organisation | Yes | No | Child of Intervention |
| Session | Organisation | Yes | Yes | Child of Event |
| Person | Organisation | Yes | No | Shared reference data |
| SessionAttendance | Organisation | No | No | Junction table |
| Feedback | Organisation | Yes | Yes | Includes anonymous submissions |
| Issue | Organisation | Yes | Yes | Safety-relevant data |
| CorrectiveAction | Organisation | Yes | Yes | Compliance-relevant |
| InterventionStakeholder | Organisation | No | No | Junction table |
| InterventionEquipment | Organisation | No | No | Reference data |
| InterventionBarrier | Organisation | No | No | Reference data |
| Tag | Organisation | No | No | Reference data |
| InterventionTag | Organisation | No | No | Junction table |
| Evaluation | Organisation | Yes | No | Assessment data |
| ChangeImplemented | Organisation | Yes | No | QI data |
| AppLog | Organisation | No | No | Application telemetry |
| FlowLog | Organisation | No | No | Flow telemetry |

### Relationship Cascade Rules

| Parent | Child | Delete | Assign | Share | Reparent |
|--------|-------|--------|--------|-------|---------|
| Intervention | Event | Cascade | Cascade | Cascade | Cascade |
| Event | Session | Cascade | Cascade | Cascade | Cascade |
| Session | SessionAttendance | Cascade | NoCascade | NoCascade | NoCascade |
| Session | Feedback | Cascade | NoCascade | NoCascade | NoCascade |
| Session | Issue | RemoveLink | NoCascade | NoCascade | NoCascade |
| Issue | CorrectiveAction | Cascade | NoCascade | NoCascade | NoCascade |
| Intervention | InterventionStakeholder | Cascade | NoCascade | NoCascade | NoCascade |
| Intervention | InterventionEquipment | Cascade | NoCascade | NoCascade | NoCascade |
| Intervention | InterventionBarrier | Cascade | NoCascade | NoCascade | NoCascade |
| Intervention | InterventionTag | Cascade | NoCascade | NoCascade | NoCascade |
| Intervention | Evaluation | Cascade | NoCascade | NoCascade | NoCascade |
| Intervention | ChangeImplemented | Cascade | NoCascade | NoCascade | NoCascade |

---

## Appendix B: Power Automate Flow Specifications

### Flow 1: CEIR - Session Created - Generate Survey URL

**Trigger:** Dataverse — When a row is added (Session table)
**Actions:**
1. Get parent Event and Intervention for context
2. Generate HMAC token: `HMAC-SHA256(sessionId, environmentVariable('redi_SurveySecret'))`
3. Compose survey URL: `{SurveyServeFlowUrl}?sid={sessionId}&t={hmacToken}`
4. Generate QR code data (via a simple QR API or inline SVG generation)
5. Update Session row: set `surveyurl`, `surveyqrdata`, `surveyactive = true`

### Flow 2: CEIR - HTTP GET - Serve Survey Form

**Trigger:** HTTP Request (GET)
**Input:** Query parameters `sid`, `t`
**Actions:**
1. Validate HMAC token against `sid`
2. Get Session row; check `surveyactive = true`
3. Get parent Event name for display
4. Compose HTML form (inline CSS, Fluent-inspired styling, mobile-responsive)
5. Return 200 with HTML body (Content-Type: text/html)
**Error responses:** 400 (bad token), 404 (inactive/not found), 500 (unexpected)

### Flow 3: CEIR - HTTP POST - Process Survey Response

**Trigger:** HTTP Request (POST)
**Input:** Form-encoded body (rating, relevance, confidence, recommend, comment, sid, token)
**Actions:**
1. Parse form body
2. Validate HMAC token
3. Validate input (required fields present, ratings in range)
4. Create Feedback row in Dataverse (anonymous, source = 'QR Survey')
5. Return 200 with thank-you HTML page

### Flow 4: CEIR - Scheduled - Corrective Action Reminders

**Trigger:** Recurrence (daily, 08:00 AEST)
**Actions:**
1. List CorrectiveActions where status IN (Pending, In Progress) AND duedate <= today + 3
2. For each: get parent Issue and Intervention
3. If overdue: update status to 'Overdue'
4. Compose reminder email (grouped by Intervention)
5. Send to Intervention owner
6. Log to FlowLog

### Flow 5: CEIR - HTTP Trigger - Generate Report

**Trigger:** HTTP Request (POST, authenticated)
**Input:** JSON body `{ interventionId: string }`
**Actions:**
1. Get Intervention + all child records (events, sessions, attendance, feedback, issues, actions, evaluations, changes, stakeholders, equipment, barriers)
2. Compute aggregates (total participants, avg ratings, response rates)
3. Populate Word template
4. Convert to PDF
5. Save to SharePoint
6. Email to owner + stakeholders
7. Update Intervention: set report URL, report date
8. Return 200 with report URL

### Flow 6: CEIR - Weekly - Survey URL Health Check

**Trigger:** Recurrence (weekly, Monday 07:00 AEST)
**Actions:**
1. Get Survey Serve flow URL from Environment Variable
2. HTTP GET with test parameters
3. If response != 200: send alert email to admin
4. Log result to FlowLog

---

## Appendix C: UI Wireframe Specifications

### Dashboard Layout

```
┌─────────────────────────────────────────────────────────────┐
│  CEIR — Clinical Education Impact Registry          [user] │
├──────────┬──────────────────────────────────────────────────┤
│          │                                                   │
│  NAV     │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐│
│          │  │ Active   │ │ Sessions│ │ Feedback│ │ Open   ││
│ Dashboard│  │ Intv: 3  │ │ MTD: 12 │ │ Avg: 4.2│ │ Actions││
│ Interv.  │  │          │ │         │ │ ★★★★☆  │ │ ⚠ 5    ││
│ Issues   │  └─────────┘ └─────────┘ └─────────┘ └────────┘│
│ Reports  │                                                   │
│ Settings │  OPEN INTERVENTIONS                               │
│          │  ┌───────────────────────────────────────────────┐│
│          │  │ ● Cardiac Arrest Team Training    [Active]    ││
│          │  │   Cardiac Clinic · 3 sessions · 2 issues     ││
│          │  ├───────────────────────────────────────────────┤│
│          │  │ ● MET Call Simulation             [Planning]  ││
│          │  │   ED · 0 sessions · 0 issues                 ││
│          │  ├───────────────────────────────────────────────┤│
│          │  │ ● Medication Safety In-Service    [Followup]  ││
│          │  │   Ward 7B · 4 sessions · 1 issue             ││
│          │  └───────────────────────────────────────────────┘│
│          │                                                   │
│          │  PENDING ACTIONS                                  │
│          │  ┌───────────────────────────────────────────────┐│
│          │  │ ⚠ Replace defib pads (Ward 7B)    Due: 3 Feb ││
│          │  │ ⚠ Update MET call protocol        Due: 10 Feb││
│          │  └───────────────────────────────────────────────┘│
└──────────┴──────────────────────────────────────────────────┘
```

### Intervention Detail (Execution Phase)

```
┌─────────────────────────────────────────────────────────────┐
│  ← Back    Cardiac Arrest Team Training          [Active]   │
├─────────────────────────────────────────────────────────────┤
│  [ Planning ] [Events & Sessions] [Issues] [Followup] [📄] │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  EVENTS                                          [+ Event]  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Cardiac Arrest Sim — High Fidelity          [3 sessions]││
│  │ ┌─────────────────────────────────────────────────────┐ ││
│  │ │ Session 1 · 15 Jan · Sim Centre · 8 pax  [Done] ✓  │ ││
│  │ │ ★★★★☆ (4.3 avg, 7 responses)    [📱 QR] [👁 View] │ ││
│  │ ├─────────────────────────────────────────────────────┤ ││
│  │ │ Session 2 · 22 Jan · Ward · 6 pax        [Done] ✓  │ ││
│  │ │ ★★★★★ (4.8 avg, 5 responses)    [📱 QR] [👁 View] │ ││
│  │ ├─────────────────────────────────────────────────────┤ ││
│  │ │ Session 3 · 5 Feb · Sim Centre · 0 pax [Scheduled] │ ││
│  │ │ No feedback yet                  [📱 QR] [✏ Edit]  │ ││
│  │ └─────────────────────────────────────────────────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ BLS Skills Station                          [1 session] ││
│  │ ...                                                     ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Session QR Code Display

```
┌───────────────────────────────────────┐
│                                        │
│    Session 1 — 15 Jan 2026            │
│    Cardiac Arrest Sim                  │
│                                        │
│    ┌──────────────────────┐           │
│    │                      │           │
│    │    [QR CODE IMAGE]   │           │
│    │                      │           │
│    │                      │           │
│    └──────────────────────┘           │
│                                        │
│    Scan to provide feedback            │
│                                        │
│    [Full Screen]  [Download PNG]       │
│                                        │
│    Survey: ● Active  [Deactivate]     │
│    Responses: 7                        │
│                                        │
└───────────────────────────────────────┘
```

---

*End of Implementation Plan — CEIR v1.0*
