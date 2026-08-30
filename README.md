# WhatsApp AI Automation — Meta Cloud API + n8n

A proof-of-concept WhatsApp ordering assistant built with the **Meta WhatsApp Business Cloud API**, **n8n**, **Google Gemini**, and **Google Sheets**.

The system receives WhatsApp messages through a webhook, uses an AI Agent to understand the conversation, checks product stock, looks up existing orders, saves new orders, and sends replies back through the WhatsApp Cloud API.

> Built as a technical assessment for an AI Automation Intern role.

## Demo

**Video:** https://drive.google.com/file/d/1HFsx8P3xSWKu9URH--vdEvoJ53u1h8ZU/view?usp=sharing

## Features

- WhatsApp Cloud API message receiving and sending
- n8n webhook-based automation
- Google Gemini-powered AI Agent
- Conversation memory using the customer's WhatsApp number
- Stock lookup from Google Sheets
- Existing-order lookup from Google Sheets
- New-order creation in Google Sheets
- Meta WhatsApp message-template workflow
- Text-message filtering
- Dynamic sender/recipient handling
- Documented phone + Cloud API co-existence approach and limitation

## Architecture

```text
Customer on WhatsApp
        |
        v
Meta WhatsApp Cloud API
        |
        v
n8n WhatsApp Trigger
        |
        v
Text Message Filter
        |
        v
Google Gemini AI Agent
   |       |       |
   |       |       +--> save_order -> Google Sheets
   |       +----------> find_order -> Google Sheets
   +------------------> check_stock -> Google Sheets
        |
        v
WhatsApp Cloud API
        |
        v
Customer receives reply
```

## Tech Stack

- **Meta WhatsApp Business Cloud API** — messaging layer
- **n8n** — workflow automation and webhook handling
- **Google Gemini** — language model / AI Agent
- **Google Sheets** — stock and order data store

## Repository Files

```text
workflow.json
    Main n8n workflow for receiving, processing and replying to WhatsApp messages.

template-workflow.json
    Example workflow for sending an approved WhatsApp message template.

README.md
    Project documentation and implementation notes.

docs/coexistence.md
    Phone + Cloud API co-existence research and limitation notes.
```

## Google Sheets Structure

### `check_stock`

| Product | Quantity | Status |
|---|---:|---|
| Chicken Thali | 20 | Available |
| Veg Thali | 10 | Available |

### `Orders`

| Customer Name | Phone | Item | Quantity | Description | Status |
|---|---|---|---:|---|---|

### Workflow tools

| Tool | Purpose | Operation |
|---|---|---|
| `check_stock` | Check current product stock | Read rows |
| `find_order` | Find an existing order | **Get Row(s)** |
| `save_order` | Save a confirmed new order | Append row |

> **Important:** `find_order` must use **Get Row(s)**, not Append. `save_order` is the node that uses Append.

## Setup

### 1. Meta WhatsApp Cloud API

1. Create/configure a Meta app with the WhatsApp product.
2. Get the WhatsApp Business Account ID and Phone Number ID.
3. Configure an access token with the permissions required by the WhatsApp Cloud API.
4. Configure the n8n WhatsApp Trigger webhook in the Meta dashboard.
5. Subscribe the application to WhatsApp message events.

Do not commit access tokens, app secrets, or other credentials to this repository.

### 2. n8n

1. Import `workflow.json` into n8n.
2. Reconnect your own WhatsApp, Google Sheets, and Google Gemini credentials.
3. Confirm that the three Google Sheets tools point to the correct spreadsheet.
4. Confirm that `find_order` is configured for **Get Row(s)**.
5. Save and activate the workflow.

### 3. Google Sheets

Create the `check_stock` and `Orders` sheets using the column structures shown above, then connect the Google Sheets credential in n8n.

## Workflow Logic

```text
WhatsApp Trigger
      |
      v
If: message type == text
      |
      v
AI Agent
  |   |   |
  |   |   +--> save_order (append)
  |   +------> find_order (Get Row(s))
  +----------> check_stock (read)
      |
      v
Send WhatsApp Message
```

### Message filtering

WhatsApp webhooks can contain message events as well as delivery/read/status events. The workflow checks the message type and only passes text messages to the AI Agent.

### Conversation memory

The session key is the customer's WhatsApp number:

```text
{{ $('WhatsApp Trigger').item.json.messages[0].from }}
```

This keeps each customer's conversation context separate.

The current demo uses n8n Simple Memory. For production, durable shared memory such as PostgreSQL or Redis would be more appropriate.

### Dynamic WhatsApp reply

The workflow takes the sender and recipient information from the incoming webhook instead of hardcoding a customer number. This makes the workflow reusable for different conversations.

## WhatsApp Message Templates

WhatsApp Business messaging uses approved message templates when a business needs to initiate or continue a conversation outside the applicable customer-service window.

`template-workflow.json` is included as a separate example for sending an approved template. The template name and language must match the template configured in WhatsApp Manager.

## Phone + Cloud API Co-existence

### What it means

WhatsApp phone + Cloud API co-existence allows a business to keep using the WhatsApp Business App while also connecting the same business number to Cloud API automation, subject to Meta's eligibility and onboarding requirements.

This is different from the normal Cloud API phone-number registration flow.

### What happened in this assessment

The number was initially registered through the standard Cloud API flow. That is not the co-existence onboarding path, so the intended co-existence setup could not be completed within the assessment environment/time window.

The co-existence onboarding path requires the appropriate Meta Embedded Signup / partner setup and eligibility. Business verification and app/solution review requirements can also apply depending on the onboarding route.

Because those Meta-side requirements could not be completed within the 48-hour assessment window, co-existence is documented here rather than claimed as a working feature.

See [`docs/coexistence.md`](docs/coexistence.md) for the detailed explanation and proposed human-handoff logic.

### Human handoff concept

If a human replies from the WhatsApp Business App while automation is active, the workflow should detect the relevant echo/sent event and temporarily suppress automated replies for that conversation. This prevents the human and AI from responding at the same time.

This handoff logic was not presented as fully implemented because the co-existence onboarding itself was not completed.

## Known Limitations

- Phone + Cloud API co-existence was not fully configured in the assessment environment.
- Simple Memory is not durable across restarts/workers.
- Webhook signature verification should be added for a production deployment.
- Idempotency/duplicate-message handling should be added to prevent duplicate order creation when webhook retries occur.
- The demo currently focuses on text messages.
- Credentials are intentionally not included in the exported workflow.

## Security Notes

Never commit the following to GitHub:

- Meta access tokens
- Meta app secrets
- n8n credentials
- Google OAuth credentials
- Google service-account private keys
- `.env` files containing secrets

Use environment variables or the credential manager provided by n8n instead.

## Assessment Summary

This project demonstrates a complete working automation path for:

**WhatsApp → Webhook → AI Agent → Google Sheets tools → WhatsApp reply**

The phone + Cloud API co-existence limitation is explicitly documented rather than presented as completed functionality.
