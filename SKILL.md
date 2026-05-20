---
name: telnyx-agentic
description: Use when the user asks an agent to make, debug, or automate Telnyx outbound AI voice calls with menu navigation, DTMF, transcript polling, retries, and proof logs. Best for agentic calling, IVR handling, voicemail attempts, call-control recovery, and turning one-off Telnyx API snippets into reliable operations.
---

# Telnyx Agentic

Use this skill for Telnyx-powered outbound voice work where the agent must actually place calls, navigate phone menus, recover from Telnyx/API errors, poll transcripts, and prove what happened.

## Core Rules

- Do not place an outbound call, send SMS, submit a form, or contact a third party unless the current user explicitly asked to send, fire, call, or otherwise execute that outbound action.
- Never write API keys, tokens, cookies, SIP passwords, or full secrets into repos, skill files, logs, screenshots, or final responses.
- Use `TELNYX_API_KEY` from the user's existing global environment first. If absent, check common local env loaders such as `~/.env` without printing secret values.
- If a call can spend money, verify account balance and outbound destination restrictions before retrying.
- Treat Telnyx conversation transcripts, call states, recordings, and saved JSON logs as proof. Do not claim a call reached a person, left voicemail, or completed from the initial queued response alone.
- Prefer small direct shell/Python/curl commands for one-off operations. Create repo-local reusable wrappers only when the user wants repeatable agentic calling.

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
  -d '{"whitelisted_destinations":["US","CA","ES","HK"]}'
```

Preserve existing countries and add only the required destination. Re-read the profile after patching.

## Assistant Pattern

For a serious call, create or update a task-specific AI assistant rather than reusing an unrelated sales/support assistant.

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

Include `send_dtmf`, `hangup`, and if available `skip_turn` tools. `skip_turn` is important for IVR systems with long announcements because premature DTMF often causes invalid-entry loops.

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

## When A Reusable Wrapper Is Appropriate

If the user wants repeatable agentic calling, create a small repo-local script that:

- reads `TELNYX_API_KEY` from env
- verifies/preserves outbound profile whitelist
- creates or updates a task-specific assistant
- places calls to primary and backup numbers
- retries transient Telnyx polling errors
- saves conversation messages and call state to local proof logs
- prints a concise result object

Keep the wrapper task-specific and keep secrets out of source control.

## Final Response Shape

End with proof, not vibes:

- `SHIPPED` only if a human was reached, voicemail was left, or the requested setup was completed and verified.
- `CHECKPOINT` if calls were placed and proof collected but no useful human/voicemail outcome happened.
- Include exact call numbers, conversation ids, call durations, DTMF/menu choices, and saved log paths.
- Name the exact blocker if one remains, for example destination whitelist, insufficient funds, no payment method, unauthenticated API key, no TeXML app, no live audio bridge, or IVR dead end.
