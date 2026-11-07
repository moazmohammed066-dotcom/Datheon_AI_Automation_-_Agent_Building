# Task 2: Autonomous Support & Email Triage (AI Draft Replies)

An [n8n](https://n8n.io) workflow that watches a support inbox, classifies each incoming email with an LLM, looks up relevant answers in an Airtable knowledge base, and writes a **reply draft** in Gmail. A human reviews every draft. **Nothing is ever sent automatically.**

```
Gmail Trigger → Extract fields → Skip automated senders → Loop per email
   → AI classify → Airtable KB search → AI draft reply
   → Gmail draft (in the same thread) → Apply labels
```

## What it does

1. **Polls Gmail every minute** for unread inbox messages that are not yet labelled `AI-Triaged` and not sent by you (`-label:AI-Triaged -from:me`).
2. **Skips automated senders** (`no-reply`, `noreply`, `donotreply`, `mailer-daemon`, or an empty sender).
3. **Classifies** each email (GPT-4o-mini via OpenRouter, temperature 0.1) into structured JSON:
   - `category`: Technical / Billing / Sales / General Inquiry
   - `urgency`: High / Medium / Low
   - `sentiment`: Positive / Neutral / Frustrated / Angry
   - `summary`, `search_query`, `reasoning`
4. **Searches the Airtable FAQ table** (top 3 matches) using the category and keywords from the classifier.
5. **Drafts a reply** (GPT-4o via OpenRouter, temperature 0.4) that uses only facts found in the knowledge base. If nothing relevant is found, the draft politely says a specialist will follow up instead of guessing.
6. **Creates a Gmail draft** in the original thread with a `Re:` subject.
7. **Labels the thread**: `AI-Triaged` and `Needs-Review` on every email, plus `AI-Urgent` when urgency is High or sentiment is Angry.

### Built-in safeguards

- Drafts only, never auto-send.
- The classifier prompt treats email content as untrusted data and ignores instructions embedded in it (prompt-injection resistance).
- The drafter is told not to invent company facts (locations, prices, policies, timelines).
- If the email contains Arabic characters, the reply language is set to Arabic; otherwise the draft matches the customer's language.
- The `AI-Triaged` label is excluded by the trigger query, so an email is never processed twice.
- If the knowledge-base lookup fails, the workflow continues with "no match" and the drafter falls back to the "a specialist will follow up" reply.
- If a required Gmail label is missing, the flow stops with a clear error that tells you to run the setup branch.

## Requirements

- An n8n instance (cloud or self-hosted) with the **LangChain / AI nodes** available, including the OpenRouter chat model node.
- A **Gmail** account (OAuth2 credential) that can read mail, create drafts, and manage labels.
- An **OpenRouter** API key.
- An **Airtable** base with a knowledge-base table and a Personal Access Token that can read its records.

## Setup

### 1. Import the workflow

In n8n: **Workflows → Import from file** and choose [`autonomous-support-email-triage.json`](./autonomous-support-email-triage.json).

This copy is sanitised for public sharing: credentials, instance and workflow IDs, and webhook IDs were removed, and the Airtable IDs are placeholders.

### 2. Create your credentials

Add and assign these credentials on the nodes that need them:

| Credential | Used by |
|---|---|
| Gmail OAuth2 | New Support Email, Create Reply Draft, Get Gmail Labels, Apply Triage Labels, and the three `Create Label …` nodes |
| OpenRouter API | GPT-4o-mini Classifier, GPT-4o Drafter |
| Airtable Personal Access Token | Search Knowledge Base |

### 3. Build the knowledge base in Airtable

Create a table (the workflow expects it to be called **FAQ**) with these fields:

| Field | Type | Notes |
|---|---|---|
| `Question` | Single line text | The topic or question |
| `Answer` | Long text | The approved resolution |
| `Category` | Single select | `Technical`, `Billing`, `Sales`, `General Inquiry`, or `All` |
| `Keywords` | Single line text | Comma-separated search keywords |

Entries with `Category = All` match every email category.

Example rows:

| Question | Answer | Category | Keywords |
|---|---|---|---|
| How do I reset my password? | Use the "Forgot password" link on the login page. A reset email arrives within a few minutes. | Technical | password, reset, login, forgot |
| How do I get an invoice? | Invoices are available under Account → Billing → Invoices. | Billing | invoice, receipt, billing, payment |
| What are your support hours? | Our support team answers emails on business days. | All | hours, support, contact, response |

Then open the **Search Knowledge Base** node and select your own base and table (the IDs in the file are placeholders: `appDEMO0000000000` and `tblDEMO0000000000`).

### 4. Create the Gmail labels (one time)

Click **Execute workflow** on the **Setup: Create Gmail Labels** manual trigger. It creates `AI-Triaged`, `Needs-Review`, and `AI-Urgent`. The main flow stops with a clear error if any of these labels are missing.

### 5. Check the models

- **GPT-4o-mini Classifier** is preset to `openai/gpt-4o-mini`.
- **GPT-4o Drafter** has no model preset in the file, so open the node and choose a model (for example `openai/gpt-4o`).

### 6. Test, then activate

1. Send a test email to the connected inbox from another address.
2. Run the workflow manually, or wait for the trigger, and check that a draft appears in the thread with the expected labels.
3. Also test: an email whose answer is **not** in the knowledge base (expect the "specialist will follow up" draft), an angry email (expect `AI-Urgent`), and an email containing "ignore previous instructions" (expect it to be classified normally).
4. When you are happy with the results, **publish / activate** the workflow.

## Customization

- **Polling interval and inbox filter**: edit the *New Support Email* node.
- **Categories, urgency rules, and tone**: edit the system prompts in *Classify Email* and *Draft Reply*. If you change categories, also update the enum in *Classification Schema* and the Airtable `Category` values.
- **Senders to skip**: edit the conditions in *Skip Automated Senders*.
- **Knowledge-base search**: the `filterByFormula` in *Search Knowledge Base* matches keywords against `Question`, `Keywords`, and `Answer`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Error "Missing Gmail labels" | Run the **Setup: Create Gmail Labels** branch once. |
| Every draft says "a specialist will follow up" | The Airtable base/table placeholders were not replaced, the credential cannot read the table, or the FAQ has no matching `Category` / `Keywords`. |
| The Drafter node errors about the model | Choose a model in the **GPT-4o Drafter** node. |
| The same email is processed repeatedly | The `AI-Triaged` label does not exist or was not applied. Check the label setup. |

## Project structure

```
.
├── README.md
└── autonomous-support-email-triage.json   # n8n workflow export (sanitised)
```

## Notes

- Always review drafts before sending. LLM output can be wrong.
- Emails are sent to your LLM provider (OpenRouter and the model vendor). Check that this fits your privacy and compliance requirements.
