# Task 4: End-to-End Multi-Agent Enterprise Operations System

An [n8n](https://n8n.io) system that automates the full client lifecycle: **intake → AI-drafted contract (PDF) → Stripe payment → verified onboarding in Slack → weekly executive digest**.

Three specialist AI agents (contract, onboarding, executive reporting) are coordinated by a **supervisor router**. Everything that touches money, records, or messaging is done by deterministic nodes. The agents only produce text.

Built with n8n, OpenRouter (`openai/gpt-4o-mini`), Airtable, Stripe, Gmail, Google Drive, Slack, and Gotenberg.

## System overview

The main workflow contains three independent paths, each with its own trigger:

| Path | Trigger | Result |
| :-- | :-- | :-- |
| **1. Intake & contract** | `POST` webhook | Client record, Stripe Checkout link, AI-drafted contract as a PDF, archived in Drive and emailed to the client |
| **2. Payment & onboarding** | Stripe `checkout.session.completed` | Payment validated, client marked `Paid / Active`, private Slack channel created with a welcome message |
| **3. Weekly digest** | Schedule (Mondays, 09:00) | Metrics computed from Airtable, AI summary posted to `#management-updates` |

## Architecture

```
[ Client Intake Webhook ] ──┐
[ Stripe Webhook Event  ] ──┼──►  End-to-End Operations workflow (05)
[ Monday 9:00 Schedule  ] ──┘                 │
                                              │  Execute Workflow (task_type + payload)
                                              ▼
                              Enterprise Operations Supervisor (04)
                               deterministic router + guards
                          ┌──────────────┼───────────────┐
                          ▼              ▼               ▼
                   Agent A (01)    Agent B (02)     Agent C (03)
                   Contract        Onboarding       Executive Reporting
```

### Supervisor contract

The main workflow never calls an agent directly. It calls the supervisor with a `task_type` and a `payload`:

| `task_type` | `payload` | Routed to | Returns |
| :-- | :-- | :-- | :-- |
| `contract` | `clientName`, `clientEmail`, `serviceTier`, `amount`, `currency`, `agreementDate` | Agent A | `{ text }` |
| `onboarding` | `clientName`, `serviceTier`, `amountPaid`, `currency`, `recordId`, `recordStatus`, `paymentVerified` | **Onboarding Guard**, then Agent B | `{ welcomeSummary, onboardingSteps[] }` |
| `executive_digest` | `metrics` (object) | Agent C | `{ text }` |
| anything else | n/a | Rejected with an `Unsupported task_type` error | n/a |

The supervisor never starts payments, writes to Airtable, sends email, or posts to Slack.

### The agents

| Agent | Job | Output checks |
| :-- | :-- | :-- |
| **A: Contract Agent** | Drafts a plain-text service agreement (under 450 words) from the intake data. Uses `[placeholders]` for anything not provided and ends with a "not legal advice" line. | Output must be non-empty |
| **B: Client Onboarding Agent** | Writes a short welcome summary and exactly 5 onboarding steps as JSON (kickoff, access, scope, milestones, communication). Never judges payment. | JSON must parse and contain a summary and at least 3 steps |
| **C: Executive Reporting Agent** | Writes a Slack-formatted digest with four fixed sections (Headline, Key Metrics, Risks and Opportunities, Priority Actions). | At most 180 words, all four sections present, and **every number in the text must exist in the supplied metrics** |

If Agent C fails its checks, the main workflow posts the raw metrics instead of an AI summary, so a hallucinated figure never reaches leadership.

## Path details

### Path 1: Client intake and contract

1. **Client Intake Webhook** receives the request and **Normalize Intake Payload** cleans it. **Valid Payload?** requires a name, a valid email, a service tier, and an amount greater than 0. Otherwise it answers `400`.
2. **Create Client Record** adds the client to Airtable with `Status = Pending Payment`.
3. **Create Stripe Checkout Session** creates a one-time payment session (the amount is converted to cents). The Airtable record ID is stored in the session metadata, and **Save Checkout Session ID** writes the session ID back to Airtable.
4. **Supervisor: Contract Agent** returns the contract text. **Build Contract HTML** turns it into a styled A4 page.
5. **Gotenberg** converts the HTML to PDF, and the result is validated as a real PDF. The PDF is saved to **Google Drive**, and the same file continues on.
6. **Email Contract & Payment Link** sends the PDF and the Stripe link through Gmail.

Responses: `200` success, `400` invalid payload, `502` when the checkout session exists but the contract PDF or email failed (the response still contains the `checkout_url`).

### Path 2: Payment verification and onboarding

1. **Stripe Checkout Completed** fires. The payment must have `payment_status = paid`, otherwise a "Payment Not Confirmed" alert is posted and nothing else happens.
2. **Find Client Record** looks up the client by the Airtable record ID in the metadata, or by email with status `Pending Payment`. No match posts an **Unmatched Payment** alert.
3. **Already Active?** ignores duplicate Stripe events for a client who is already `Paid / Active`.
4. **Payment Matches Record?** requires all of the following: status is `Pending Payment`, the amount paid equals the stored amount, the currency is `USD`, and the stored Stripe session ID equals the event's session ID. Any mismatch posts a validation alert and the client is **not** activated.
5. **Mark Client Paid / Active** updates Airtable. **Supervisor: Onboarding Agent** runs behind the **Onboarding Guard** (`paymentVerified = true` and `recordStatus = "Paid / Active"`).
6. A **private Slack channel** named `client-<client-name>-<record-suffix>` is created, the team is invited, and a Block Kit welcome message with the agent's onboarding steps is posted. If channel creation fails, an alert goes to `#management-updates`.

### Path 3: Weekly executive digest

Every Monday at 09:00 (in the n8n instance's timezone) the workflow reads all clients from Airtable and computes these metrics in code: `totalClients`, `activeClients`, `totalRevenue`, `pendingPaymentsCount`, `pendingPaymentsValue`, `newSignupsThisWeek`, `newSignupsPaid`, `newSignupsRevenue`, `activeClientsByTier`, and `newClientNames`. Agent C turns them into a short summary that is posted to `#management-updates`.

### Error handling

- PDF conversion, Drive archiving, channel creation, unmatched or invalid payments, and unconfirmed payments each post a Slack alert to `#management-updates`.
- A **Workflow Error Trigger** posts any other failure (workflow name, failed node, error, execution link). To enable it, open the main workflow's **Settings → Error workflow** and select the workflow itself.

## Repository contents

| File | Description |
| :-- | :-- |
| [`01-agent-a-contract-agent.json`](./01-agent-a-contract-agent.json) | Sub-workflow that drafts the service agreement |
| [`02-agent-b-client-onboarding-agent.json`](./02-agent-b-client-onboarding-agent.json) | Sub-workflow that writes the onboarding plan |
| [`03-agent-c-executive-reporting-agent.json`](./03-agent-c-executive-reporting-agent.json) | Sub-workflow that writes the weekly digest |
| [`04-enterprise-operations-supervisor.json`](./04-enterprise-operations-supervisor.json) | Router that dispatches to the agents and applies guards |
| [`05-end-to-end-multi-agent-enterprise-operations-system.json`](./05-end-to-end-multi-agent-enterprise-operations-system.json) | Main workflow with the webhook, Stripe, schedule, PDF, email, Drive, and Slack logic |

The numeric prefixes show the **import order**, because each workflow references the ones before it.

## Requirements

- **n8n** with Execute Workflow nodes and the LangChain / AI nodes.
- **Gotenberg** for HTML-to-PDF. For local use:
  ```bash
  docker run --rm -p 3000:3000 gotenberg/gotenberg:8
  ```
  The workflow calls `http://host.docker.internal:3000/forms/chromium/convert/html`. Change this URL in the **Gotenberg: HTML to PDF** node if your Gotenberg runs elsewhere.
- **Credentials** in n8n:
  - **Airtable** Personal Access Token with read/write on the clients base.
  - **Stripe** API key (use **test mode** first). n8n registers the Stripe webhook automatically when the workflow is activated, so the instance must be reachable from the internet.
  - **Gmail** OAuth2 (sends the contract email).
  - **Google Drive** OAuth2 (archives the PDF).
  - **Slack** bot token. Typical scopes: `chat:write`, `groups:write`, `channels:manage`, `channels:read`, `users:read`. Invite the bot to `#management-updates`.
  - **OpenRouter** API key.

## Setup

### 1. Airtable table

Create a table with these fields:

| Field | Type |
| :-- | :-- |
| `Client Name` | Single line text |
| `Email` | Email |
| `Service Tier` | Single line text |
| `Amount` | Number |
| `Status` | Single select: `Pending Payment`, `Paid / Active` |
| `Stripe Session ID` | Single line text |

### 2. Slack

Create a channel named `#management-updates` and invite the bot.

### 3. Import the workflows (in this order)

1. `01-agent-a-contract-agent.json`
2. `02-agent-b-client-onboarding-agent.json`
3. `03-agent-c-executive-reporting-agent.json`
4. `04-enterprise-operations-supervisor.json`
5. `05-end-to-end-multi-agent-enterprise-operations-system.json`

### 4. Replace the placeholders

The files are sanitised, so wire them together and fill in your own values:

| Placeholder | Where to change it |
| :-- | :-- |
| `REPLACE_WITH_AGENT_A_WORKFLOW_ID`, `..._AGENT_B_...`, `..._AGENT_C_...` | The three **Call Agent ...** nodes in workflow **04**. Pick the imported agent workflow in each. |
| `REPLACE_WITH_SUPERVISOR_WORKFLOW_ID` | The three **Supervisor: ...** Execute Workflow nodes in workflow **05**. Pick the imported supervisor. (The sticky note named "Multi-agent architecture" lists the same IDs for reference.) |
| `appXXXXXXXXXXXXXX` / `tblXXXXXXXXXXXXXX` | The 5 Airtable nodes in workflow **05**. |
| `REPLACE_WITH_SLACK_USER_ID` | **Prepare Channel & Team Config** (`accountManagerSlackId` and `onboardingLeadSlackId`) and the `userIds` of **Invite Team Members**. Use real Slack member IDs (they start with `U` or `W`). Until then the welcome message falls back to generic role names. |
| `https://example.com/payment-success` / `payment-cancelled` | **Normalize Intake Payload**: set your real success and cancel URLs. |
| Gotenberg URL | **Gotenberg: HTML to PDF**, if not running on `host.docker.internal:3000`. |

Then assign your credentials on every node that needs one.

### 5. Test the full flow

1. **Intake.** Send a request (use the test URL `/webhook-test/...` while the workflow is open in the editor, and `/webhook/...` once it is active):
   ```bash
   curl -X POST "https://<your-n8n-host>/webhook/enterprise-ops/client-intake" \
     -H "Content-Type: application/json" \
     -d '{
       "client_name": "Acme Corp",
       "client_email": "client@example.com",
       "service_tier": "Premium",
       "amount": 1500
     }'
   ```
   Expect a `200` response with `airtable_record_id` and `checkout_url`, a new `Pending Payment` row, a PDF in Drive, and an email with the PDF and the Stripe link.
2. **Payment.** Open the `checkout_url` and pay with Stripe's test card `4242 4242 4242 4242`. Expect the row to become `Paid / Active` and a new private `client-...` Slack channel with the welcome message.
3. **Digest.** Run the **Every Monday 9:00 AM** branch manually and check `#management-updates`.
4. **Guards.** Confirm that an invalid payload returns `400`, and that sending the Supervisor `task_type = onboarding` without a verified payment is rejected.

## Limitations

- The currency is fixed to **USD** (in **Normalize Intake Payload** and the payment check).
- Slack members to invite are fixed IDs in the workflow. Use a lookup or a config table if you need different people per client.
- "Active" and "pending" in the weekly metrics are detected from the `Status` text, and "new signups" use Airtable's record creation time.
- Clients created before the `Stripe Session ID` field was filled in fail payment validation by design.
- The contract is an AI draft for business review and is **not legal advice**.
- Client data is sent to your LLM provider (OpenRouter) and to the other connected services.

## Project structure

```
task-4-multi-agent-enterprise-ops/
├── README.md
├── 01-agent-a-contract-agent.json
├── 02-agent-b-client-onboarding-agent.json
├── 03-agent-c-executive-reporting-agent.json
├── 04-enterprise-operations-supervisor.json
└── 05-end-to-end-multi-agent-enterprise-operations-system.json
```
