# Webinar Lead Automation (GoHighLevel + n8n)

Two n8n workflows that run the email side of a webinar funnel on top of GoHighLevel (GHL): an instant registration confirmation, and a scheduled follow-up sequence. Both keep their state on the GHL contact, so a contact never gets the same email twice, even across retries and restarts.

> **In production for a client.** The live system runs on the client's own domain and isn't linked here. This repository is the build record: the full source, with client data, credentials and URLs removed.

## Workflows

### Webinar Confirmation Email — `n8n/webinar-confirmation-email.json`

`POST /webhook/webinar-confirmation` (fired by a GHL workflow on registration)
→ validate the payload, resolve the contact in GHL, and check a **guard field** plus a 24-hour dedup window, so retries and double submissions are ignored
→ Gmail confirmation with the join link
→ write the guard fields back to the contact (`confirmation_sent`, `sequence_step = 0`, `next_send_at`) → notify the team
→ respond `200 confirmation_sent`, `500` if Gmail failed (so GHL can retry), or an "ignored" response for invalid requests and duplicates.

### Webinar Lead Follow-up Sequence — `n8n/webinar-lead-follow-up-sequence.json`

Runs every 30 minutes. It finds contacts whose `next_send_at` has passed, skips anyone tagged `unsubscribed`, `do-not-email` or sequence-complete, and sends each the next step (replay, resource or booking link) through Gmail, capped at 40 per run to stay under Gmail's sending limits. It then writes the new step and next send time back to GHL.

## Setup

1. Import both files into n8n and connect a **Gmail** credential.
2. In GHL, create four single-line text custom fields (confirmation sent, sequence step, next send at, last sent at).
3. In each `CONFIG — EDIT ME` block, set the location ID, the field IDs, and the webinar, replay and booking links. Supply the GHL private integration token (`GHL_TOKEN`) from a secure source and never commit a real one.
4. Point a GHL workflow's webhook action at the confirmation workflow's production URL.
