---
name: telnyx-agentic
description: Use when the user asks an agent to make, debug, or automate Telnyx outbound AI voice calls, single-call or campaign-scale, with AI assistants, Missions tracking, DTMF/IVR navigation, transcript polling, retries, and proof logs. Best for agentic calling, 10-100 lead outreach campaigns, voicemail attempts, call-control recovery, and turning one-off Telnyx snippets into reliable operations.
---

# Telnyx Agentic

Use this skill for Telnyx-powered outbound voice work where the agent must actually place calls, navigate phone menus, run multi-lead campaigns, recover from Telnyx/API errors, poll transcripts, and prove what happened.

## Core Rules

- Do not place an outbound call, send SMS, submit a form, or contact a third party unless the current user explicitly asked to send, fire, call, or otherwise execute that outbound action.
- Never write API keys, tokens, cookies, SIP passwords, or full secrets into repos, skill files, logs, screenshots, or final responses.
- Use `TELNYX_API_KEY` from the user's existing global environment first. If absent, check common local env loaders such as `~/.env` without printing secret values.
- If a call can spend money, verify account balance and outbound destination restrictions before retrying.
- Treat Telnyx conversation transcripts, call states, recordings, and saved JSON logs as proof. Do not claim a call reached a person, left voicemail, or completed from the initial queued response alone.
- Prefer small direct shell/Python/curl commands for one-off operations. Create repo-local reusable wrappers only when the user wants repeatable agentic calling.
- For 2+ leads, use campaign discipline: a lead list, a call plan, per-lead status, retry windows, transcript proof, and a final outcome table. For 10+ leads, use Telnyx Missions or an equivalent state ledger.
- Create or update only campaign-specific assistants. Do not mutate unrelated shared assistants, production routing, TeXML apps, SIP connections, or phone-number assignments unless the user explicitly asked for that infrastructure change.

## Choose The Right Telnyx Tool

Do not blindly use every Telnyx feature. Use the smallest set that fits the job.

| Need | Use |
| --- | --- |
| One-off AI phone call | AI Assistant + TeXML AI call + conversations/call state |
| 10-100 lead outbound campaign | Missions + campaign-specific AI Assistant + TeXML AI calls + events + conversations |
| IVR or phone menu | `send_dtmf`, `skip_turn`, transcript polling, call state |
| Human-in-the-loop live audio | SIP/WebRTC or a Telnyx voice bridge; normal TeXML AI calls are autonomous, not live agent audio |
| Offline audio analysis | STT/recording tools when recordings are available |
| Scripted voice assets | TTS only when you need pre-generated audio or testing |
| Product/company knowledge inside calls | Assistant instructions first; RAG/embeddings only when there is a real document corpus |
| SMS follow-up | Messaging only after explicit user permission to send SMS |
| Network/tunnel exposure | Network tooling only for webhook servers or SIP/WebRTC bridges |

Missions are for tracking and orchestration; they do not replace call execution. A campaign still needs assistants, calls, transcript polling, and proof.

## Campaign Mode: 20 Business Collaboration Calls

If the user says something like "call these 20 businesses and propose collaboration", do not run twenty isolated calls from memory. Build a campaign run.

Minimum campaign ledger:

```json
{
  "campaign_name": "business-collaboration-outreach",
  "objective": "Propose collaboration and identify interested decision makers",
  "from_number": "+15551234567",
  "assistant_id": "assistant_x",
  "leads": [
    {
      "lead_id": "lead_001",
      "business_name": "Example Co",
      "phone": "+15557654321",
      "contact_name": "",
      "status": "pending",
      "attempts": [],
      "outcome": "",
      "next_step": ""
    }
  ]
}
```

Required lead fields:

- `business_name`
- `phone` in E.164 format
- `campaign_angle` or short reason for collaboration
- optional `contact_name`, `website`, `locale`, `notes`, and `do_not_call`

Execution rules:

- Deduplicate phone numbers before calling.
- Skip leads missing a valid E.164 number.
- Call in small batches; default concurrency is 1 unless the user explicitly wants parallel calling and the Telnyx account/channel limits support it.
- Use retry policy: one initial call, one later retry for no-answer/busy/temporary IVR failure, no retry after explicit refusal or do-not-call.
- Save every attempt: queued response, conversation id, call state, transcript, outcome classification, and next step.
- Classify outcomes as `interested`, `callback_requested`, `send_info`, `not_interested`, `wrong_number`, `voicemail_left`, `no_answer`, `ivr_dead_end`, `failed`, or `needs_human_review`.
- Adjust the assistant only at campaign boundaries or after clear evidence from transcripts. Do not rewrite instructions after every single call unless a repeated failure pattern appears.
- If the assistant makes a systematic mistake, pause further calls, patch the assistant prompt, record the change, and resume.

For a 20-business campaign, the agent should create or reuse:

- one campaign-specific assistant
- one Mission or equivalent run ledger
- one plan step for setup
- one plan step for calling
- one plan step for follow-up/outcome review
- one event per call attempt
- one final summary table

## Missions Mode

Use Missions for multi-call campaigns, long-running work, or any job where recovery/audit matters.

Useful Mission API surface from Telnyx:

- create/list/update/delete missions
- list runs for a mission
- add steps to a run plan
- log events for a mission run
- get step details
- link/list/unlink Telnyx agents for missions or runs when using Telnyx-managed agents

Inline examples:

```bash
# Create a mission
curl -sS -X POST "https://api.telnyx.com/v2/ai/missions" \
  -H "Authorization: Bearer $TELNYX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"business-collaboration-outreach"}'

# List missions
curl -sS "https://api.telnyx.com/v2/ai/missions" \
  -H "Authorization: Bearer $TELNYX_API_KEY"

# Add steps to a run plan once mission_id/run_id are known
curl -sS -X POST "https://api.telnyx.com/v2/ai/missions/$MISSION_ID/runs/$RUN_ID/plan/steps" \
  -H "Authorization: Bearer $TELNYX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"steps":[{"step_id":"setup","description":"Prepare assistant and lead list","sequence":1},{"step_id":"calls","description":"Call leads and collect outcomes","sequence":2},{"step_id":"review","description":"Summarize outcomes and follow-ups","sequence":3}]}'

# Log a call event
curl -sS -X POST "https://api.telnyx.com/v2/ai/missions/$MISSION_ID/runs/$RUN_ID/events" \
  -H "Authorization: Bearer $TELNYX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"type":"status_change","summary":"Called lead_001; voicemail left; follow up by email","metadata":{"lead_id":"lead_001","outcome":"voicemail_left"}}'
```

Telnyx SDK/docs sometimes expose similar mission-agent linking endpoints with slightly different path names. Prefer the currently working API response over memory. If an endpoint returns 404/422, inspect the installed Telnyx SDK docs or official docs and retry the current path; do not invent success.

## Fast Setup Checks

Run these before calling:

```bash
test -n "$TELNYX_API_KEY" && echo TELNYX_API_KEY_PRESENT
curl -sS https://api.telnyx.com/v2/balance \
  -H "Authorization: Bearer $TELNYX_API_KEY"
curl -sS https://api.telnyx.com/v2/phone_numbers \
  -H "Authorization: Bearer $TELNYX_API_KEY"
curl -sS https://api.telnyx.com/v2/outbound_voice_profiles \
  -H "Authorization: Bearer $TELNYX_API_KEY"
curl -sS https://api.telnyx.com/v2/ai/assistants?limit=20 \
  -H "Authorization: Bearer $TELNYX_API_KEY"
curl -sS https://api.telnyx.com/v2/texml_applications \
  -H "Authorization: Bearer $TELNYX_API_KEY"
```

Report only resource ids, phone numbers already relevant to the task, status, destination whitelist, and whether required values exist. Do not echo secrets.

## Outbound Destination Whitelist

If Telnyx returns `D13` or says the dialed country is not whitelisted, inspect and patch the outbound voice profile:

```bash
PROFILE_ID="<outbound_voice_profile_id>"
curl -sS "https://api.telnyx.com/v2/outbound_voice_profiles/$PROFILE_ID" \
  -H "Authorization: Bearer $TELNYX_API_KEY"

curl -sS -X PATCH "https://api.telnyx.com/v2/outbound_voice_profiles/$PROFILE_ID" \
  -H "Authorization: Bearer $TELNYX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"whitelisted_destinations":["US","CA","ES","<DESTINATION_COUNTRY_CODE>"]}'
```

Preserve existing countries and add only the required destination. Re-read the profile after patching.

## Assistant Pattern

For a serious call or campaign, create or update a task-specific AI assistant rather than reusing an unrelated sales/support assistant.

The assistant instructions should include:

- who the agent is calling on behalf of
- the exact goal and success criteria
- the facts the agent may say
- what to ask if a human answers
- what voicemail to leave
- when to hang up
- DTMF rules for known menu choices
- rules to wait silently through long announcements
- rules not to repeat rejected digits
- outcome classification rules
- what data to collect before the call ends
- what must be escalated to the user

Include `send_dtmf`, `hangup`, and if available `skip_turn` tools. `skip_turn` is important for IVR systems with long announcements because premature DTMF often causes invalid-entry loops.

For business collaboration campaigns, the assistant prompt should include:

- the offer in one sentence
- the target audience
- acceptable outcomes, such as interest, callback, request for info, refusal, wrong number
- the user's contact details or follow-up method, if the user supplied them
- a short voicemail version
- a rule to stop politely after refusal
- a rule not to overpromise pricing, partnership terms, legal terms, or availability unless provided

Safe adjustment loop:

1. Start with one test call or a small batch of 2-3 leads.
2. Read transcripts and classify errors.
3. Adjust only the assistant instructions that caused the observed error.
4. Continue with the next batch.
5. Keep a changelog of prompt adjustments in the campaign ledger or Mission events.

## Place An AI Call

Use the TeXML AI call endpoint when a TeXML application is connected to the sending number:

```bash
TEXML_APP_ID="<texml_app_id>"
ASSISTANT_ID="<assistant_id>"
FROM="+15551234567"
TO="+15557654321"

curl -sS -X POST "https://api.telnyx.com/v2/texml/ai_calls/$TEXML_APP_ID" \
  -H "Authorization: Bearer $TELNYX_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"From\":\"$FROM\",\"To\":\"$TO\",\"AIAssistantId\":\"$ASSISTANT_ID\"}"
```

The returned `queued` status only proves Telnyx accepted the outbound attempt. It is not proof of contact.

For campaigns, wrap each call with:

1. pre-call event: lead id, target number, assistant id, attempt number
2. TeXML AI call request
3. conversation polling
4. call state fetch
5. outcome classification
6. mission/local ledger update

## Poll Proof

After the call starts, poll conversations and match the call by `metadata.to`, `metadata.from`, `metadata.assistant_id`, and `metadata.call_control_id`.

```bash
curl -sS "https://api.telnyx.com/v2/ai/conversations?limit=20" \
  -H "Authorization: Bearer $TELNYX_API_KEY"

CONVERSATION_ID="<conversation_id>"
curl -sS "https://api.telnyx.com/v2/ai/conversations/$CONVERSATION_ID/messages?limit=100" \
  -H "Authorization: Bearer $TELNYX_API_KEY"

CALL_CONTROL_ID="<call_control_id>"
curl -sS "https://api.telnyx.com/v2/calls/$CALL_CONTROL_ID" \
  -H "Authorization: Bearer $TELNYX_API_KEY"
```

Save a local proof log when the workflow matters:

```bash
mkdir -p call_logs
curl -sS "https://api.telnyx.com/v2/ai/conversations/$CONVERSATION_ID/messages?limit=100" \
  -H "Authorization: Bearer $TELNYX_API_KEY" \
  > "call_logs/telnyx-conversation-$CONVERSATION_ID.json"
```

Summarize the exact transcript evidence:

- menu prompt heard
- DTMF digit or tool call sent
- whether a human answered
- whether voicemail was offered
- whether a voicemail was actually recorded
- call duration and `is_alive`
- conversation id and saved log path
- campaign lead id and attempt number when applicable
- outcome label and next action

## Outcome Table

For campaigns, final output should include a compact table:

| Lead | Phone | Attempts | Outcome | Evidence | Next step |
| --- | --- | ---: | --- | --- | --- |
| Example Co | +15557654321 | 1 | interested | conversation id + quoted line | send info |

Also include counts:

- total leads
- called
- reached human
- voicemail left
- interested/callback/send-info
- not interested
- failed/no answer/IVR dead end
- needs human review

## IVR Menu Lessons

Use these hard-won rules when navigating phone menus:

- Language menus can be handled immediately when explicit, for example `for English press 2`.
- After language selection, wait through long organization announcements before pressing more digits.
- Do not infer an option from a partial transcript fragment. Wait until the menu says the option and digit.
- If the menu says `preliminary advice press 1` and `applications for registration press 2`, choose the branch that matches the user's actual goal.
- If an option returns `input incorrect`, do not repeat it unless the transcript clearly changed.
- If an operator is unavailable, look for explicit voicemail wording and the digit for voicemail.
- Do not treat a directory lookup as success. If it asks for last-name letters and no contact name is known, exit or hang up rather than guessing.
- Use `skip_turn` or silence while announcements are still playing.
- Hang up stuck calls; then retry with improved instructions. Do not let an agent loop indefinitely.

## Recovery From Tool Failures

Telnyx API polling can fail from local DNS or remote disconnects while the call itself continues. When polling fails:

1. Re-list recent conversations with `curl`.
2. Find the newest conversation matching the target number and assistant.
3. Fetch messages by conversation id.
4. Fetch call state by `call_control_id`.
5. Hang up if the call is still alive and trapped in IVR.
6. Save the recovered transcript and state.

Do not say the call failed just because local polling crashed.

For campaigns, after recovery save an event to the Mission or local ledger with the failure type and recovery action.

## When A Reusable Wrapper Is Appropriate

If the user wants repeatable agentic calling or campaign execution, create a small repo-local script that:

- reads `TELNYX_API_KEY` from env
- verifies/preserves outbound profile whitelist
- creates or updates a task-specific assistant
- places calls to primary and backup numbers
- retries transient Telnyx polling errors
- saves conversation messages and call state to local proof logs
- prints a concise result object
- supports a lead-list JSON/CSV input for campaigns
- supports dry-run validation without calling
- supports batch size, retry policy, and pause/resume
- logs per-lead events to Missions or a local state file

Keep the wrapper task-specific and keep secrets out of source control.

## Final Response Shape

End with proof, not vibes:

- `SHIPPED` only if a human was reached, voicemail was left, or the requested setup was completed and verified.
- `CHECKPOINT` if calls were placed and proof collected but no useful human/voicemail outcome happened.
- Include exact call numbers, conversation ids, call durations, DTMF/menu choices, and saved log paths.
- For campaigns, include the outcome table and counts.
- Name the exact blocker if one remains, for example destination whitelist, insufficient funds, no payment method, unauthenticated API key, no TeXML app, no live audio bridge, or IVR dead end.
