# Datheon AI Automation & Agent Building

Deliverables for the **AI Automation & Agent Building Internship at Datheon**.

This repository contains three production-style automation projects built with [n8n](https://n8n.io), LLMs (via OpenRouter), and business tools such as Airtable, Slack, Gmail, and Stripe. Each project is a set of importable n8n workflow files plus its own README that explains the design, setup, and testing.

> The internship syllabus asks for at least 2 or 3 of the 4 tasks. This repository completes **Tasks 1, 2, and 4**.

## Projects

| Task | Project | What it does | Main tools | Folder |
| :-: | :-- | :-- | :-- | :-- |
| 1 | **Multi-Source Lead Routing & AI Qualification** | Takes inbound form leads, validates and de-duplicates them, scores them with a hybrid AI and rule-based rubric, saves them to a CRM, and alerts sales. | Tally, OpenRouter, Airtable, Slack | [`task-1-lead-routing-ai-qualification`](./task-1-lead-routing-ai-qualification) |
| 2 | **Autonomous Support & Email Triage** | Reads a support inbox, classifies each email, searches a knowledge base, and writes a reply **draft** for human review. Never auto-sends. | Gmail, OpenRouter, Airtable | [`task-2-support-email-triage`](./task-2-support-email-triage) |
| 4 | **End-to-End Multi-Agent Enterprise Operations System** | Client intake, AI-drafted contract (PDF), Stripe payment, verified onboarding in Slack, and a weekly executive digest. Three specialist AI agents are coordinated by a supervisor router. | Stripe, Airtable, Slack, Gmail, Google Drive, Gotenberg, OpenRouter | [`task-4-multi-agent-enterprise-ops`](./task-4-multi-agent-enterprise-ops) |

## Repository structure

```
.
├── README.md
├── .gitignore
├── task-1-lead-routing-ai-qualification/
│   ├── README.md
│   └── multi-source-lead-routing-ai-qualification.json
├── task-2-support-email-triage/
│   ├── README.md
│   └── autonomous-support-email-triage.json
└── task-4-multi-agent-enterprise-ops/
    ├── README.md
    ├── 01-agent-a-contract-agent.json
    ├── 02-agent-b-client-onboarding-agent.json
    ├── 03-agent-c-executive-reporting-agent.json
    ├── 04-enterprise-operations-supervisor.json
    └── 05-end-to-end-multi-agent-enterprise-operations-system.json
```

## Common design principles

These ideas are shared across all three projects:

- **Human in the loop where it matters.** Task 2 only creates drafts. Task 4 only onboards a client after the payment passes several deterministic checks.
- **AI writes, code decides.** Scores, routing, payment validation, and figure checks are done by deterministic code. The LLM produces text and judgments that are then validated.
- **Untrusted input.** Emails and form submissions are treated as data, never as instructions (prompt-injection resistant prompts plus input sanitisation).
- **Failure paths are explicit.** Failed steps raise Slack alerts or flag the record for review instead of failing silently.

## How to use any workflow

1. Open your n8n instance and go to **Workflows → Import from file**.
2. Select the `.json` file (for Task 4, follow the import order in its README).
3. Create credentials for each service and assign them to the nodes that need them.
4. Replace the placeholder values described below.
5. Test with sample data, then activate the workflow.

### Placeholders to replace

The workflow files are **sanitised for public sharing**: credentials, instance and workflow IDs, webhook IDs, and account-specific IDs were removed. Replace these placeholders with your own values:

| Placeholder | Meaning | Where |
| :-- | :-- | :-- |
| `appXXXXXXXXXXXXXX`, `tblXXXXXXXXXXXXXX` | Airtable base ID and table ID | Tasks 1, 2, 4 |
| `REPLACE_WITH_SLACK_CHANNEL_ID` | Slack channel that receives lead alerts | Task 1 |
| `REPLACE_WITH_TALLY_FORM_ID` | Your Tally form | Task 1 |
| `REPLACE_WITH_SLACK_USER_ID` | Slack member ID of the account manager / onboarding lead | Task 4 |
| `REPLACE_WITH_*_WORKFLOW_ID` | n8n IDs of the imported sub-workflows | Task 4 |

Each project README lists the exact nodes to edit.

## Security notes

- No API keys, tokens, or passwords are stored in this repository.
- Never commit exported workflows without checking them first. Exports can contain credential IDs, instance IDs, and webhook URLs.
- Customer emails and form content are sent to your LLM provider (OpenRouter and the model vendor). Check that this fits your privacy and compliance requirements before using real data.

## Author

Built as part of the **AI Automation & Agent Building Internship** at **Datheon**.
