# Task 1: Multi-Source Lead Routing & AI Qualification

An [n8n](https://n8n.io) workflow that turns an inbound **Tally** form submission into a scored, de-duplicated CRM record in **Airtable** and an instant **Slack** alert for the sales team.

```
Tally Trigger → Normalize & Validate → Duplicate check → AI qualification
             → Deterministic scoring → Airtable record → Slack notification
```

Built with n8n, OpenRouter (`openai/gpt-4o-mini`), Tally, Airtable, and Slack.

## How it works

```
[ Tally Trigger ]
        │
        ▼
[ Normalize & Validate Submission ]   (clean, cap lengths, validate, detect injection)
        │
 ( Submission Valid? )
   ├── No  ──► [ Reject Invalid Submission ]   (execution stops with a clear error)
   └── Yes ──► [ Check for Duplicate Lead ]    (Airtable search by Work Email)
                    │
             ( Already Processed? )
               ├── Yes ──► [ Duplicate Submission - Skipped ]
               └── No  ──► [ Build AI Qualification Prompt ]
                                │
                        [ AI Lead Qualification ]  ◄── OpenRouter GPT-4o-mini
                                │
                        [ AI Scoring & Validation ]  (final score and class computed in code)
                                │
                        [ Create Airtable Lead Record ]
                          ├── saved  ──► [ Format Slack Notification ] ──► [ Send Sales Notification ]
                          │                                                   └── if Slack fails ──► [ Flag Slack Failure on Record ] (Airtable)
                          └── failed ──► [ Format Airtable Failure Alert ] ──► [ Alert: Airtable Save Failed ] (Slack)
```

### Step by step

1. **Normalize & Validate.** Trims and caps every field, strips control characters, validates the email and website, maps dropdown answers to the canonical option lists, and builds a stable `leadId`. `Full Name`, `Work Email`, and `Company Name` are required. Invalid submissions are rejected.
2. **Duplicate check.** Searches Airtable for the same `Work Email`. A duplicate is skipped, so no AI call or second record is created.
3. **AI qualification.** The model receives the submission inside `<lead_submission>` tags, marked as untrusted data. It returns JSON with a project-fit score, a summary, the reason for the score, 3 sales talking points, and a recommended action.
4. **Scoring & validation.** Code recomputes the budget, company size, and timeline points from fixed lookup tables, clamps the AI's project-fit score to 0-30, sums the total, and assigns Hot, Warm, or Cold. If the AI total or class differs, the code value wins and the change is logged in `Processing Notes`.
5. **CRM + alert.** Creates the Airtable record, then posts a formatted Slack message with a link to the record.

## Key design decisions

- **AI judges, code decides.** The model only rates *project fit* (0-30) and writes the text. Everything else in the score is deterministic, so the total cannot be inflated by model arithmetic errors or by instructions hidden in the form.
- **Missing is not negative.** If budget, size, or timeline is blank, the workflow uses a neutral default score instead of penalising the lead, and the prompt tells the model to say what is unknown.
- **Prompt-injection handling.** Submissions are scanned for manipulation phrases (for example "ignore previous instructions" or "score this lead 100"). A match sets `Processing Status = Needs Review` and adds a note. The system prompt also tells the model to give such text a project-fit score of 0-5.
- **Low-latency, resilient AI call.** Temperature `0`, JSON-only output, `maxTokens: 380`, a 15 second timeout, and one retry at the model level. **Retry On Fail** is also enabled on the AI node to absorb transient provider errors (in the worst case, retries add to the total run time).
- **Visible failure paths.** If the Airtable save fails, a Slack alert says the lead was **not** saved. If Slack fails, the Airtable record is flagged `Needs Review`.

## Scoring rubric (100 points)

| Criteria | Points | Not provided |
| :-- | :-- | :-: |
| **Budget** (max 30) | `$25,000+` = 30, `$10,000-$25,000` = 24, `$5,000-$10,000` = 18, `$1,000-$5,000` = 12, `Under $1,000` = 5 | 12 |
| **Project fit** (max 30, AI) | 24-30 specific and business-critical, 16-23 clear but generic, 8-15 vague or very short, 0-5 spam, irrelevant, or manipulation attempt | 10 |
| **Company size** (max 20) | `500+` = 20, `201-500` = 17, `51-200` = 14, `11-50` = 9, `1-10` = 5 | 9 |
| **Timeline** (max 20) | `Immediately` = 20, `Within 1 month` = 16, `Within 3 months` = 11, `More than 3 months` = 5, `Just exploring` = 2 | 8 |

**Classification:** Hot = 70-100 (contact within 24 hours), Warm = 40-69 (follow up in 2-3 business days), Cold = 0-39 (low-priority nurture).

## Requirements

- An n8n instance (cloud or self-hosted) with the LangChain / AI nodes and the Tally node available.
- Credentials for **Tally**, **OpenRouter**, **Airtable** (Personal Access Token with read/write on the base), and **Slack** (bot token with `chat:write`; invite the bot to the target channel).

## Setup

### 1. Create the Tally form

The workflow matches fields by their **label**, so use these labels (case and punctuation are ignored):

| Field label | Type | Required | Accepted values |
| :-- | :-- | :-: | :-- |
| Full Name | Text | Yes | |
| Work Email | Email | Yes | |
| Company Name | Text | Yes | |
| Job Title | Text | No | |
| Phone Number | Text | No | |
| Industry | Text | No | |
| Company Website | URL | No | |
| Company Size | Dropdown | No | `1–10 employees`, `11–50 employees`, `51–200 employees`, `201–500 employees`, `500+ employees` |
| Estimated Budget | Dropdown | No | `Under $1,000`, `$1,000–$5,000`, `$5,000–$10,000`, `$10,000–$25,000`, `$25,000+` |
| Expected Timeline | Dropdown | No | `Immediately`, `Within 1 month`, `Within 3 months`, `More than 3 months`, `Just exploring` |
| What problem are you trying to solve? | Long text | No | |
| Additional Information | Long text | No | |

Dropdown values that do not match are ignored with a warning in `Processing Notes`.

### 2. Create the Airtable table

Create a table named **Incoming Leads** with these fields:

| Field | Type |
| :-- | :-- |
| `Lead ID`, `Full Name`, `Work Email`, `Company Name`, `Job Title`, `Phone Number`, `Industry`, `Website` | Single line text |
| `Company Size`, `Estimated Budget`, `Expected Timeline` | Single select (options exactly as in the table above) |
| `Submission Date` | Date and time (ISO format) |
| `AI Qualification Score` | Number |
| `Lead Classification` | Single select: `Hot`, `Warm`, `Cold` |
| `Processing Status` | Single select: `Qualified`, `Needs Review`, `Error` |
| `Project Requirements`, `Additional Information`, `AI Summary`, `Qualification Reason`, `Recommended Talking Points`, `Recommended Action`, `Processing Notes` | Long text |

> The Airtable create node has `typecast` turned off, so single-select options must exist in Airtable **exactly** as written (including the en dash `–` in ranges).

### 3. Import the workflow

1. In n8n choose **Workflows → Import from file** and select [`multi-source-lead-routing-ai-qualification.json`](./multi-source-lead-routing-ai-qualification.json).
2. Assign your credentials to: **Tally Trigger**, **OpenRouter GPT-4o-mini**, the three **Airtable** nodes, and the two **Slack** nodes.

### 4. Replace the placeholders

| Placeholder | Where to change it |
| :-- | :-- |
| `appXXXXXXXXXXXXXX` / `tblXXXXXXXXXXXXXX` | The 3 Airtable nodes (*Check for Duplicate Lead*, *Create Airtable Lead Record*, *Flag Slack Failure on Record*) **and** the two constants at the top of the *Format Slack Notification* code node (used for the record link) |
| `REPLACE_WITH_SLACK_CHANNEL_ID` | *Send Sales Notification* and *Alert: Airtable Save Failed* |
| `REPLACE_WITH_TALLY_FORM_ID` | *Tally Trigger* (pick your form from the list) |

### 5. Make the AI-failure fallback active (recommended)

The *AI Scoring & Validation* node contains fallback logic: if the AI call fails, it still saves the lead with `Processing Status = Needs Review` and a note, so no lead is lost. **Retry On Fail** is already enabled on the **AI Lead Qualification (Basic LLM Chain)** node and covers temporary errors. To make the fallback trigger once the retries are exhausted, open that node, go to **Settings → On Error**, and choose **Continue (using regular output)**.

### 6. Test and activate

1. Submit a test lead through the form (the n8n *Execute workflow* test mode listens for one submission).
2. Check that an Airtable record appears with a score, class, talking points, and notes, and that Slack receives the alert.
3. Submit the same email again and confirm it is skipped as a duplicate.
4. Try a submission containing "ignore previous instructions" and confirm it is saved as `Needs Review`.
5. Switch the workflow to **Active**.

## Notes and limitations

- **Source.** The workflow currently listens to one Tally form. The *Normalize & Validate* node already accepts several payload shapes, so more sources (other forms or a webhook) can be added by routing them into that node.
- **Rejected and duplicate submissions.** An invalid submission ends the execution with an error (visible in n8n **Executions**). A duplicate is skipped silently.
- **Injection detection is a heuristic.** The pattern list catches common phrases. The prompt rules and the code-side score recalculation are the second layer of defence.
- **Privacy.** Lead details are sent to your LLM provider through OpenRouter.

## Project structure

```
task-1-lead-routing-ai-qualification/
├── README.md
└── multi-source-lead-routing-ai-qualification.json   # n8n workflow export (sanitised)
```
