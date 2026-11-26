---
name: investigate
description: Multi-turn SRE Agent investigation for a PagerDuty incident
---

Investigate a PagerDuty incident with the SRE Agent in a multi-turn conversation. Follow these steps in order. Do not silently degrade — if a required tool is unavailable, stop and explain.

The user will provide an incident ID (e.g. `Q3AHWAR33KUK3G` or `#971097`), or ask you to list active incidents first.

## Pre-flight

Call `pagerduty-browse_incidents` with `action: "list"` and `limit: 1` to confirm the PagerDuty MCP server is connected.

If the call fails with a transport, auth, or "tool not found" error, STOP and tell the user:

```
PagerDuty MCP server is not available. This skill requires it.

A PagerDuty account admin must install the GitHub Copilot CLI app once for your account (one-time step):
https://app.pagerduty.com/oauth-apps/PEMCLUK
If it is not installed, sign-in fails after login with an error that the app is not installed in the account.
Then connect the "pagerduty" MCP server (sign in when prompted) and retry.
If Copilot CLI asks for a client ID, enter https://github.com/copilot/cli/client-metadata.json
Can't get the app installed? Register your own OAuth app in PagerDuty and connect with its client ID and secret instead:
- Add the "pagerduty" MCP server yourself with `copilot mcp add --transport http pagerduty "https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents"`
- When you authenticate, Copilot asks for a client ID and client secret; enter your app's
- Copilot uses a loopback redirect on a port it picks (for example http://127.0.0.1:55453/). Register that exact redirect URL, with the port Copilot shows, in your OAuth app. If the first sign-in fails with a redirect mismatch, copy the redirect URL Copilot used, add it to the app, and retry
Details: https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server#use-an-oauth-client-id-and-client-secret
Docs: https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server/cimd-clients
```

Do NOT proceed without PagerDuty MCP.

## Step 1 — Incident resolution

Parse the user's message:

**Alphanumeric ID** (e.g. `Q3AHWAR33KUK3G`): use directly as `incident_id`. Skip to Step 2.

**Numeric or `#`-prefixed** (e.g. `#971097`): strip `#`, call `pagerduty-browse_incidents` with `action: "list"` and find the incident whose `incident_number` matches. Use its `id` field. If no match, ask the user to provide the alphanumeric ID directly.

**No ID given**: call `pagerduty-browse_incidents` with `action: "list"` and `statuses: ["triggered", "acknowledged"]`. Show the first 10 results (ID, title, service, age) and ask which one to investigate. If empty, tell the user no active incidents were found and ask them to provide an incident ID or describe what they want to investigate.

Do not call the SRE Agent until an incident ID is confirmed.

## Step 2 — Initial analysis

The SRE Agent is a two-step interaction: `pagerduty-paige_sre_agent_send_message` submits a message and returns a `conversation_id` immediately; `pagerduty-paige_sre_agent_get_response` returns the answer.

If either tool is missing ("tool not found"), STOP and tell the user: the PagerDuty SRE Agent requires PagerDuty Advance on the account and the `ai-agents` tool group on the PagerDuty MCP server. See https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server#step-1-choose-an-authorization-method

**Send.** Call `pagerduty-paige_sre_agent_send_message` with:
- `incident_id`: the resolved alphanumeric ID
- `message`: `"Summarize incident [id]: what happened, root cause, and top recommended actions."`

**AI Actions cost gate.** The SRE Agent consumes AI Actions (formerly PagerDuty Advance credits). On the first call in a conversation, do **not** pass `user_confirmed_ai_actions_cost`. The response has status `confirmation_required` and a `warning`. Show the warning to the user and ask them to reply "yes". Only if they say yes, repeat the same call with `user_confirmed_ai_actions_cost: "yes"`, and pass it on every later send in this conversation without asking again. If they decline, stop and do not call the SRE Agent again.

Store the `conversation_id` from the response and reuse it for every follow-up.

**Poll.** Call `pagerduty-paige_sre_agent_get_response` with the `conversation_id`:
- `is_complete: false` (status `working`) is expected. Call it again with the same `conversation_id` until `is_complete` is true.
- `is_complete: true` and status `idle`: the `messages` field holds the answer. Display it in full.
- `is_complete: true` and status `no_response`: the run was interrupted. Resend the same message once, then poll again. If it fails again, tell the user the SRE Agent is unavailable and stop.

If a send returns status `busy`, poll instead of resending.

## Step 3 — Conversation loop

After displaying each SRE Agent response, ask the user:
"What would you like to explore next? You can ask a follow-up question, save this conversation to a file, or say done."

**Follow-up question**: call `pagerduty-paige_sre_agent_send_message` with `incident_id`, `message=<user question>`, and `user_confirmed_ai_actions_cost: "yes"`, then poll with `pagerduty-paige_sre_agent_get_response` and the stored `conversation_id` as in Step 2. Display the result. Ask the same follow-up prompt again.

**Save**: write the full conversation transcript to `./on-call-reports/sre-[incident-id]_[YYYY-MM-DD_HH-MM].md` via `bash`. Create `./on-call-reports/` if needed. Confirm the path. Continue the loop — the user may ask more questions after saving.

**Done**: exit.
