# ElevateBox Voice Agent

An outbound AI voice sales agent. It phones a lead, holds a natural conversation in
**Telugu, Hindi, or English** (including mid-sentence code-switching), asks discovery
questions, classifies the lead as **Hot, Warm, or Cold**, and follows up on WhatsApp
through tool-calling webhooks.

## What it does

- **Voice conversation:** an OmniDimension agent handles telephony, speech-to-text,
  the LLM, and text-to-speech, with interruption handling.
- **Multilingual:** detects the caller's language from the first reply and stays in it,
  handling Telugu/English and Hindi/English mixing without breaking flow.
- **Lead qualification:** asks six discovery questions one at a time (what they sell,
  catalog size, existing site, budget, timeline, special needs) and classifies the lead
  continuously.
- **Tool calling:** custom-API tool actions call this repo's Flask webhooks.
  - **Hot:** a WhatsApp message goes out mid-call, without pausing the conversation.
  - **Warm:** the specific barrier the caller named is captured, and a callback time
    spoken by the caller is parsed and confirmed.
  - **Cold:** the call ends politely with a follow-up message.
- **Post-call follow-up:** every call ends with a WhatsApp message containing real
  context from the call, the caller's stated needs, a contact number, a resume, and an
  architecture image.

## Architecture

```
dispatch_call.py --> OmniDimension agent (telephony + STT + LLM + TTS)
                          |  custom-API tool actions
                          v
                 Flask webhooks (webhook_server.py)
                  /webhook/midcall   /webhook/followup
                          |
                          v
              WhatsApp via Meta Cloud API (whatsapp.py)
```

Details and the full call flow are in [`architecture.md`](architecture.md).

## Project structure

| File | Purpose |
| --- | --- |
| `agent_prompt.md` | Conversation design: persona, language handling, questions, classification, actions |
| `agent_setup.py` | Creates the OmniDimension agent and wires the webhook tool actions |
| `webhook_server.py` | Flask server with `/webhook/midcall`, `/webhook/followup`, `/health` |
| `whatsapp.py` | Meta Graph API sender (text, document, image) |
| `dispatch_call.py` | Triggers the outbound call |
| `.env.example` | Required configuration |

## Setup

1. Create an OmniDimension account and get an API key.
2. Set up WhatsApp access through the Meta Cloud API (phone number ID and access token).
3. Install and configure:

   ```bash
   pip install -r requirements.txt
   cp .env.example .env   # fill in the values
   ```

4. Start the webhook server and expose it on a public HTTPS URL (for example with a
   tunnel), then set `WEBHOOK_BASE_URL` to that URL:

   ```bash
   flask --app webhook_server run --port 8000
   ```

5. Create the agent, then place a call:

   ```bash
   python agent_setup.py
   python dispatch_call.py <agent_id> <from_number_id>
   ```

## Configuration

| Variable | Meaning |
| --- | --- |
| `OMNIDIM_API_KEY` | OmniDimension API key |
| `META_WA_PHONE_NUMBER_ID`, `META_WA_ACCESS_TOKEN` | Meta WhatsApp Cloud API credentials |
| `WEBHOOK_BASE_URL` | Public HTTPS URL of `webhook_server.py` |
| `RESUME_PUBLIC_URL`, `ARCHITECTURE_IMAGE_URL` | Public URLs of the attachments sent after a call |
| `TARGET_PHONE_NUMBER` | Number the agent calls |
| `PORT` | Webhook server port (default 8000) |

## Design decisions

- **OmniDimension over a hand-built pipeline.** It bundles telephony, STT, TTS, and
  interruption handling in one platform and is built for Indian-language outbound
  calling, which removes the work of stitching telephony, an STT service, an LLM, and
  a TTS service together.
- **Own the WhatsApp step.** OmniDimension's native integrations do not include sending
  a WhatsApp message with a resume and an image attached, so the agent calls this
  server through a custom-API tool action and the server controls content and
  attachments.
- **Fast webhooks.** The mid-call webhook returns quickly so the live conversation is
  never blocked.
- **Classify by action, not label.** Hot fires a message now, Warm records the named
  barrier and schedules a callback, and Cold disengages, so each label has a concrete
  consequence.
- **Secrets stay out of the repo.** All credentials come from environment variables.

## Operational notes

- WhatsApp free-form messages to a number are only deliverable inside an open customer
  conversation window; outside it, use an approved message template.
- The conversation content registered with the agent is defined in `agent_setup.py`
  and mirrors `agent_prompt.md`; keep the two in sync when editing the script.

## Tech stack

Python, Flask, OmniDimension (voice agent platform), Meta WhatsApp Cloud API,
python-dotenv, requests.
