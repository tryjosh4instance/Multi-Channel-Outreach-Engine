# Multi-Channel Outreach Workflow Engine

A lightweight outbound-lead automation system built with **n8n, Google Sheets, Gmail, and Docker**.

The system standardizes lead intake, campaign assignment, scheduled outreach, lead lifecycle management, and operational error handling while maintaining a centralized lead database.

> **Portfolio MVP:** This project demonstrates workflow orchestration, business-rule automation, scheduled outreach, database state management, duplicate prevention, lifecycle management, and operational monitoring.

---

## Overview

Sales teams often manually track leads, follow-up timing, campaign status, and outreach activity across different processes. This can result in missed follow-ups, inconsistent sales processes, outdated lead statuses, and limited visibility into campaign progress.

This project automates those repetitive processes while keeping the underlying business rules configurable and visible.

---

## Architecture

The MVP uses **four operational workflows**, plus a separate controlled-error workflow used to test the error-handling path.

```text
                         Lead Source
                      (Google Sheets)
                             │
                             ▼
              ┌──────────────────────────┐
              │ Workflow 1                │
              │ Lead Intake &             │
              │ Campaign Progression      │
              └────────────┬─────────────┘
                           │
                           ▼
                    Campaign Sequence
                           │
                           ▼
              ┌──────────────────────────┐
              │ Workflow 3                │
              │ Campaign Scheduler        │
              └────────────┬─────────────┘
                           │
                           ▼
                         Gmail
                           │
                           ▼
                    Lead Status Update
                           │
              ┌────────────┴─────────────┐
              │                          │
              ▼                          ▼
       Workflow 2                  Workflow 4
       Lead Monitoring             Error Handling &
       & Lifecycle                 Manual Intervention
              │                          │
              ▼                          ▼
       Inactive Lead               Error Log + Alert
       Management                  for Manual Review
```

---

## Workflow 1 — Lead Intake & Campaign Progression

**Purpose:** Validate and prepare new leads for campaign execution.

The workflow:

1. Detects new lead records from Google Sheets.
2. Checks for duplicate email addresses.
3. Removes duplicate entries when applicable.
4. Routes leads to the appropriate campaign.
5. Determines the campaign sequence step.
6. Sends the configured outreach email.
7. Updates the lead record with status, activity, and next action.

Duplicate prevention is based on the lead's email address.

---

## Workflow 2 — Lead Monitoring & Lifecycle

**Purpose:** Keep lead records current and prevent inactive leads from remaining in active campaign processing.

The workflow evaluates lead activity and identifies leads that have been inactive for **30+ days**.

Inactive leads are:

- Marked as `Inactive`
- Assigned the next action `Archive Lead`
- Removed from the active schedule
- Annotated for review

This keeps the active lead database focused on leads that are still progressing through the workflow.

---

## Workflow 3 — Campaign Scheduler

**Purpose:** Execute scheduled campaign steps based on each lead's `Next Action At` value.

The scheduler:

1. Retrieves leads from the database.
2. Identifies leads whose next action is due.
3. Retrieves the appropriate campaign sequence step.
4. Prepares the outreach action.
5. Sends the configured email through Gmail.
6. Records the activity timestamp.
7. Calculates the next action.
8. Advances the campaign sequence.
9. Marks the campaign as completed when the configured sequence is finished.

The campaign configuration is stored separately from the lead records, allowing sequence timing and actions to be changed without rebuilding the core workflow.

---

## Workflow 4 — Error Handling & Manual Intervention

**Purpose:** Capture workflow failures and create a structured path for human intervention.

When an n8n workflow fails, the error workflow:

1. Captures the failed workflow and node.
2. Generates an incident ID and timestamp.
3. Records the error message and execution information.
4. Stores the incident in the Google Sheets error log.
5. Sends an email alert for manual review.

This creates an operational audit trail instead of allowing workflow failures to remain invisible.

---

## Controlled Error Testing

The repository also includes a separate **Controlled Error Testing** workflow.

It intentionally triggers an n8n error so the error-handling workflow can be tested under a known failure condition.

This was used to validate the complete failure path:

```text
Controlled Failure
       ↓
Error Trigger
       ↓
Failure Record
       ↓
Google Sheets Error Log
       ↓
Manual Intervention Email
```

The controlled test is included as part of the portfolio to demonstrate that error handling was not only designed, but tested.

---

## Campaign Sequences

The MVP includes two example campaign configurations.

### Q3 Outreach

| Step | Action | Wait |
|---|---|---:|
| 1 | Send Intro Email | 0 days |
| 2 | Send Follow-up Email | 2 days |
| 3 | Send Final Email | 7 days |

Sequence ends with `Sequence Completed`.

### Enterprise Campaign

| Step | Action | Wait |
|---|---|---:|
| 1 | Send Intro Email | 0 days |
| 2 | Send Follow-up Email | 3 days |
| 3 | Send Final Email | 5 days |

Sequence ends with `Sequence Completed`.

The campaign sequence configuration is maintained separately so campaign timing and actions can be managed as data rather than hard-coded workflow logic.

---

## Data Model

The MVP uses Google Sheets as the system of record.

### Lead

Stores the prospect and current lifecycle state, including:

- Lead ID
- Name
- Company
- Job Title
- Email
- LinkedIn URL
- Phone
- Status
- Campaign
- Owner
- Last Activity
- Next Action
- Next Action At

### Campaign

Defines a reusable outreach campaign.

### Sequence Step

Defines an individual campaign action, channel, template reference, and delay.

### Outreach Activity

Represents an individual communication attempt and its delivery status.

### Workflow Error

Stores failures requiring manual intervention, including the error type, message, timestamp, and resolution status.

---

## Technology Stack

- **n8n** — Workflow orchestration and automation
- **Google Sheets** — Lead database, campaign configuration, activity tracking, and error logging
- **Gmail** — Outbound email delivery and operational alerts
- **Docker** — Local n8n deployment

---

## Key Automation Features

### Duplicate Prevention

Incoming leads are checked against existing records using email address matching before entering campaign processing.

### Campaign Routing

Leads are assigned to the appropriate campaign based on configured campaign rules.

### Scheduled Follow-ups

Campaign steps are executed according to configurable waiting periods and `Next Action At` scheduling.

### Lead Lifecycle Management

Leads that remain inactive for 30+ days are automatically moved to an inactive/archive state.

### Error Monitoring

Workflow failures are captured, logged, and surfaced for manual intervention.

### State Management

Lead status, current sequence step, last activity, and next action are updated as the campaign progresses.

---

## Testing & Validation

The MVP was tested across:

- Lead intake
- Campaign assignment
- Duplicate prevention
- Missing-contact validation
- Initial outreach
- Follow-up sequences
- Sequence progression
- Campaign completion
- Activity timestamps
- Lead inactivity detection
- Lifecycle/archive state
- Workflow failure handling
- Error logging
- Manual intervention alerts

The acceptance-test summary reports that the implemented functional requirements passed, including the controlled scheduled failure test.

---

## MVP Scope

The current version intentionally does **not** include:

- Automated LinkedIn message sending
- CRM integrations
- AI-generated outreach copy
- Real-time reply sentiment analysis
- SMS or WhatsApp outreach
- Advanced analytics dashboards
- Multi-user role management
- Cloud-hosted production deployment
- Automatic lead enrichment

These are planned as potential future enhancements rather than requirements of the MVP.

---

## Error Handling Strategy

The broader technical design defines different handling paths for common failure types:

| Error Type | Strategy |
|---|---|
| Temporary API Failure | Retry up to 3 times |
| Network Timeout | Retry up to 3 times |
| Validation Error | Manual correction |
| Authentication Failure | Manual intervention |

Errors are designed to be logged with the lead/workflow context, error type, message, timestamp, and resolution status.

---

## Security Considerations

The production design calls for:

- Storing credentials using n8n credential management rather than hardcoding them
- Restricting access to the lead database
- Limiting sensitive lead information to authorized users
- Avoiding API keys, tokens, and passwords in workflow logs
- Restricting manual database changes
- Protecting the n8n environment with authentication and appropriate network controls

---

## Repository Contents

```text
Multi-Channel-Outreach-Engine/
│
├── README.md
├── Workflow 1 - Lead Intake & Campaign Progression - GITHUB.json
├── Workflow 2 - Lead Monitoring & Lifecycle - GITHUB.json
├── Workflow 3 - Campaign Scheduler - GITHUB.json
├── Workflow 4 - Error Handling _ Manual Intervention - GITHUB.json
└── Controlled Error Testing - GITHUB.json
```

The GitHub workflow exports are sanitized versions of the working n8n workflows. Environment-specific credentials and identifiers should be configured separately when importing them into an n8n instance.

---

## Future Enhancements

### Version 1.1

- Gmail reply detection
- Slack notifications for failed workflows
- Configurable follow-up timing rules

### Version 2.0

- CRM integration
- AI-assisted lead prioritization
- LinkedIn task automation and activity tracking
- Supabase or PostgreSQL database

### Long-Term Roadmap

- Multi-channel campaign orchestration
- AI-powered lead scoring
- Web-based campaign dashboard
- Analytics and conversion reporting
- Multi-user administration and permissions

---

## Project Status

**Proof of Concept / MVP**

The system has been validated with representative test leads, campaign sequences, lifecycle scenarios, and a controlled scheduled failure. The acceptance testing covered the four-workflow MVP architecture and reported all 12 functional requirements as implemented. 
