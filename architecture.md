# Architecture

```
[Trigger: dispatch_call.py -> outbound call API]
              |
              v
      [OmniDimension Agent]
   (telephony + STT + LLM + TTS,
    Hindi / Telugu / English, interruption-aware)
              |
   +----------+-----------+
   |                      |
[Discovery]          [Language detect
 budget / products /    + code-switch
 timeline / features     handling]
   |
   v
[Classification: Hot / Warm / Cold]
   |
   +--- Hot ---> [Mid-call action: custom API tool
   |              -> POST /webhook/midcall
   |              -> WhatsApp (Meta Cloud API) while call is live]
   |
   +--- Warm --> [Scheduler: parse spoken time ->
   |              book callback (Calendar / Cal.com)]
   |
   +--- Cold --> [Log + disengage politely]
              |
              v
      [Call ends]
              |
              v
   [POST /webhook/followup
    -> post-call WhatsApp: context from the call +
       resume + phone number + architecture image]
```

## Components

| Component | File | Role |
| --- | --- | --- |
| Agent design | `agent_prompt.md` | Persona, language handling, discovery questions, Hot/Warm/Cold rules, scheduling, WhatsApp content |
| Agent setup | `agent_setup.py` | Creates the OmniDimension agent and registers the two custom-API tool actions |
| Webhook server | `webhook_server.py` | Flask endpoints the agent calls: `/webhook/midcall`, `/webhook/followup`, `/health` |
| WhatsApp sender | `whatsapp.py` | Meta Graph API client for text, document and image messages |
| Call trigger | `dispatch_call.py` | Starts an outbound call to `TARGET_PHONE_NUMBER` |
