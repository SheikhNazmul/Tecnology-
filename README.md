# WhatsApp AI Automation (Meta Cloud API + n8n)

A proof of concept WhatsApp automation built with the Meta WhatsApp Business Cloud API and n8n. It receives messages through a webhook, processes them with an AI agent, reads and writes order data in Google Sheets, and replies back through the Cloud API.

Built as a technical assessment for the AI Automation Intern role.

## Demo video

<!-- EDIT: paste your video link below -->
[video link here]

## What it does

A customer sends a message on WhatsApp. The webhook fires, the workflow filters out non text events, and an AI agent handles the conversation. The agent can check stock, look up an existing order, and save a new order to a Google Sheet. The reply goes back to the same customer through the Cloud API.

The demo use case is a restaurant ordering assistant, but the messaging layer is generic.

## Stack

- Meta WhatsApp Business Cloud API
- n8n for the workflow and webhook handling
- Groq (GPT OSS 120B) as the language model
- Google Sheets as the data store

<!-- EDIT: if you go back to Gemini, change the model line above to "Google Gemini as the language model" -->

## Files

```
workflow.json           main workflow (receive, process, reply)
template-workflow.json  sends an approved Meta message template
README.md
```

## Setup

### 1. Meta app

1. Create an app at developers.facebook.com and add the WhatsApp product.
2. Note the Phone Number ID and the WhatsApp Business Account ID from the API Setup page. Both are needed later.
3. Generate an access token with `whatsapp_business_messaging` and `whatsapp_business_management`.

### 2. Webhook

1. In n8n, open the WhatsApp Trigger node and copy the production webhook URL.
2. In the Meta app dashboard go to WhatsApp then Configuration, and paste that URL as the callback URL.
3. Set a verify token. The same value must be set in the n8n WhatsApp credential.
4. Subscribe to the `messages` field.

Meta sends a GET request with a challenge parameter when you save the URL. n8n answers that automatically using the verify token stored in the credential, so no separate handler is needed.

To subscribe an app to a WhatsApp Business Account manually:

```
POST /v21.0/{WABA_ID}/subscribed_apps
```

Note that this edge lives on the WABA, not on the Phone Number ID. Using the Phone Number ID here returns error code 100 with subcode 33.

### 3. Google Sheets

Create a spreadsheet with two sheets.

`check_stock`

| Product | Quantity | Status |
|---|---|---|

`Orders`

| Customer Name | Phone | Item | Quantity | Description | Status |
|---|---|---|---|---|---|

Connect a Google Sheets credential in n8n and point the three sheet nodes at this document.

### 4. Import the workflow

1. In n8n create a new workflow and import `workflow.json`.
2. Select your own credentials in the WhatsApp, model, and Google Sheets nodes. Credentials are not included in the export.
3. Update the Sender Phone Number if you are not using the dynamic expression.
4. Save and activate.

## How the workflow is built

```
WhatsApp Trigger  ->  If  ->  AI Agent  ->  Send Message
                              |
                              +-- Chat model
                              +-- Simple Memory (conversation state)
                              +-- check_stock  (read)
                              +-- find_order   (read)
                              +-- save_order   (append)
```

### Payload handling

WhatsApp does not only send user messages to the webhook. It also sends delivery status events for sent, delivered and read. Those payloads have no `messages` array, so passing them straight to the agent throws an error.

The If node filters on:

```
{{ $json.messages[0].type }}  is equal to  text
```

Status events and non text messages such as images or voice notes fall to the false branch and stop there. Only text messages reach the agent.

### Conversation memory

Each incoming message starts a separate workflow execution. Nothing about the previous message survives on its own. The memory node is what carries the conversation forward, keyed on the sender number:

```
{{ $('WhatsApp Trigger').item.json.messages[0].from }}
```

This gives every customer their own session.

The current setup uses Simple Memory, which stores state in the n8n instance memory. That is fine for a demo but it does not survive a restart and is not shared across workers in a queue or multi main setup. For production this should be swapped for Postgres or Redis chat memory. The node itself is a drop in replacement, only the credential and connection change.

### Replying

The sender and recipient are both read from the trigger payload rather than hardcoded:

```
Sender     {{ $('WhatsApp Trigger').item.json.metadata.phone_number_id }}
Recipient  {{ $('WhatsApp Trigger').item.json.messages[0].from }}
```

So the reply always goes out from whichever number received the message, and back to whoever sent it. Adding a second number needs no change to the workflow.

## Message templates

Free form messages can only be sent inside the 24 hour window that opens when a customer messages you. Outside that window you must use a template approved by Meta.

Templates are created in WhatsApp Manager under Message Templates. Utility templates with plain text and no variables get approved fastest. Marketing templates take longer and are rejected more often.

`template-workflow.json` sends one. The template field takes the name and language code joined by a pipe:

```
template_name|en
```

The language code must match exactly what was selected when the template was created. `en` and `en_US` are different templates as far as the API is concerned.

Two limits worth knowing. The default `hello_world` template can only be sent from Meta provided test numbers, not from a registered business number. And a template with variables will fail unless the body components are supplied in the request.

## Phone and Cloud API co-existence

This is the part of the assessment I did not get fully working, so here is an honest account of what happened and what the correct path is.

### What co-existence means

Normally a WhatsApp number lives in one place. Either a person uses it in the WhatsApp Business App on a phone, or an application drives it through the Cloud API. Co-existence lets both happen on the same number at the same time. A human agent can pick up a conversation from the phone while automation handles the rest, and both see the same thread.

### What went wrong here

I registered the number through the standard Cloud API flow. That flow migrates the number to the API and logs it out of the WhatsApp Business App. By the time I understood the difference, the number was already registered the normal way.

Standard registration and co-existence onboarding are not the same path, and you cannot switch from one to the other after the fact. Co-existence has to be chosen at onboarding.

### The correct approach

Co-existence is set up through Embedded Signup, not through the standard registration screen. The business keeps using the WhatsApp Business App on the phone, and during Embedded Signup a QR code is scanned from inside the app to link the Cloud API alongside it. The number stays active in both places.

Two constraints matter once it is set up. If the client re-registers the WhatsApp Business App or moves to a new device, the Cloud API companion is offboarded automatically and has to be reconnected. And Embedded Signup versions are deprecated on a schedule, so the current version should be confirmed in Meta's documentation before building against it.

### Handling the handoff

Co-existence creates a problem the automation has to solve. If a human replies from the phone while the bot is also replying, the customer gets two answers to the same question.

Messages sent from the phone arrive at the webhook as echo events rather than incoming messages. The workflow can subscribe to those, and when one arrives for a conversation, set a flag against that session and suppress bot replies for a cooling off period. The bot resumes once the human has stopped replying. This keeps the two from talking over each other without needing a separate agent console.

I did not implement this because the co-existence setup itself did not complete, but it is the piece that would sit on top of it.

## Known limitations

- Co-existence is not configured, for the reasons above.
- Simple Memory is not durable. Postgres or Redis is needed for production.
- No signature verification on the webhook. n8n handles the verify token challenge, but the `X-Hub-Signature-256` header on incoming payloads is not checked against the app secret. A production deployment should verify it before trusting the payload.
- No duplicate handling. Meta retries a webhook if it does not get a timely response, which can save the same order twice. Storing the message id and checking it before processing would prevent this.
- Only text messages are handled. Images, voice notes and interactive replies are filtered out rather than processed.
