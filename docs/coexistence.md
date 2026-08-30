# Phone + Cloud API Co-existence

## Overview

WhatsApp phone + Cloud API co-existence allows an eligible business to keep using the WhatsApp Business App on a phone while also using Cloud API automation with the same business number.

This is different from the normal Cloud API registration flow.

## What was attempted

For this assessment, the phone number was first registered through the standard WhatsApp Cloud API flow. That path is different from co-existence onboarding, so the number could not be used to demonstrate the intended co-existence setup afterward.

The correct co-existence path uses Meta's supported Embedded Signup / partner onboarding flow and depends on the business and app meeting Meta's current eligibility and review requirements.

## Why it was not completed

The assessment was limited to 48 hours. The required Meta-side onboarding/review steps were not available for completion in that environment and timeframe.

Therefore, this repository does **not** claim that phone + Cloud API co-existence is fully implemented.

## Human handoff design

Co-existence also requires coordination between the human using the WhatsApp Business App and the automated workflow.

A practical design is:

```text
Incoming customer message
        |
        v
AI automation responds
        |
        +---- Human replies from WhatsApp Business App
                         |
                         v
                 Detect sent/echo event
                         |
                         v
                 Set handoff/session flag
                         |
                         v
                 Temporarily pause bot
                         |
                         v
                 Resume after inactivity
```

The purpose is to prevent the AI and human agent from replying to the same customer at the same time.

## Production considerations

Before using co-existence in production, verify the current Meta documentation and onboarding requirements because Meta can change supported Embedded Signup versions, eligibility requirements, and partner processes.

The automation should also consider:

- human-handoff state per customer
- inactivity/cooldown timing
- webhook retries and idempotency
- message/echo event handling
- durable session storage
- webhook signature verification

## Assessment conclusion

The core WhatsApp Cloud API automation was implemented and documented. Co-existence was researched and its correct onboarding path was identified, but the Meta-side co-existence onboarding could not be completed during the assessment window.
