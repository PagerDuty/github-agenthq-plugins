# PagerDuty for GitHub Copilot

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

PagerDuty's operational intelligence plugin for the [GitHub Copilot](https://github.com/features/copilot) AgentHQ plugin system. Brings PagerDuty incident data into your GitHub workflow.

[Install PagerDuty Github App](https://github.com/apps/pagerduty-agent-app)

## Before you start: one-time admin step

> **A PagerDuty account admin must install the GitHub Copilot CLI app once per PagerDuty account.** Until they do, nobody in the account can use this plugin.
>
> **[Install the GitHub Copilot CLI app in PagerDuty](https://app.pagerduty.com/oauth-apps/PEMCLUK)** (requires an account admin)

If the app is not installed, sign-in fails *after* you log in, with an error saying the app is not installed in the account. Send the link above to your PagerDuty admin, then try again.

With the admin step you do not need your own OAuth app. If Copilot CLI asks for a client ID, enter `https://github.com/copilot/cli/client-metadata.json`. See the [CIMD clients guide](https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server/cimd-clients) for details.

### Can't get the app installed?

Register your own OAuth app in PagerDuty and connect with its client ID and secret instead.

1. Add the `pagerduty` MCP server yourself with Copilot CLI (HTTP transport):

   ```bash
   copilot mcp add --transport http pagerduty "https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents"
   ```

2. When you authenticate, Copilot CLI asks for a client ID and client secret. Enter your app's.
3. Copilot CLI uses a loopback redirect on a port it picks (for example `http://127.0.0.1:55453/`). Register that exact redirect URL, with the port Copilot shows, in your OAuth app. If the first sign-in fails with a redirect mismatch, copy the redirect URL Copilot used, add it to the app, and retry.

See [Use an OAuth client ID and client secret](https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server#use-an-oauth-client-id-and-client-secret). Never commit the client secret to a repository.

## Capabilities

| Skill | What it does |
| --- | --- |
| [`investigate`](#investigate) | Multi-turn SRE Agent investigation for a PagerDuty incident |
| [`on-call-handoff`](#on-call-handoff) | Generate an on-call incident summary or shift handoff using PagerDuty data |
| [`pre-commit-risk-scoring`](#pre-commit-risk-scoring) | Assess pre-commit risk by correlating PagerDuty incidents with current code changes |

## Prerequisites

- A working [GitHub Copilot CLI](https://docs.github.com/copilot) installation
- Access to the PagerDuty-hosted [MCP server](https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server) (`https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents`), which the plugin configures automatically
- A PagerDuty account admin has completed the [one-time admin step](#before-you-start-one-time-admin-step)
- For `investigate`: PagerDuty Advance on your account

## Install

```bash
copilot plugin install PagerDuty/github-agenthq-plugins
```

Verify:

```bash
copilot plugin list                 # should list PagerDuty/github-agenthq-plugins
# In an interactive Copilot session:
/agent                              # confirms the PagerDuty main agent loaded
/skills list                        # confirms all skills loaded
```

## investigate

*Multi-turn SRE Agent investigation with session continuity.*

Ask the agent to investigate a PagerDuty incident:

> "Investigate incident Q3AHWAR33KUK3G"
> "What caused incident #971097?"
> "Root cause analysis for this incident"

The skill opens a multi-turn SRE Agent session:

- Resolves the incident ID (alphanumeric or numeric; lists active incidents if none provided)
- Returns a structured summary: what happened, root cause, and recommended actions
- Maintains session context across follow-up questions — deployment correlation, similar past incidents, log links — without reloading context each turn
- Optionally saves the full conversation transcript to `./on-call-reports/sre-<id>_<date>.md`

Requires PagerDuty Advance (see [Prerequisites](#prerequisites)).

## on-call-handoff

*On-call incident summary and shift handoff.*

Ask the agent for an on-call summary or shift handoff:

> "Give me an on-call handoff for the payments team"
> "Summarize the last 7 days of incidents"
> "What happened on-call this week?"

Optional arguments: `[team_name] [timeframe]` — e.g. `payments 7 days`, `platform 2 weeks`.

The skill:

- Resolves your team's escalation policies and services from PagerDuty (cached after first run in `~/on-call-handoff/config.json`)
- Fetches active incidents and all incidents in the window in parallel
- Returns a structured summary: Active, Incident Trends, Watch List, Recommended Actions
- Adapts format to timeframe: ≤14 days → handoff-focused (Active first), >14 days → trend-focused (Incident Trends first)
- Offers to save to `./on-call-reports/<team>_<date>.md` or bridge to `investigate` for deep analysis on a specific incident

## pre-commit-risk-scoring

*Pre-commit risk assessment correlated with PagerDuty incident history.*

Ask the agent about risk before committing or pushing:

> "Run a pre-commit risk check on this branch"
> "Is this change risky?"
> "Compare these changes to recent PagerDuty incidents on `<service-name>`"

The skill resolves your repo to a PagerDuty service through a fallback chain (explicit argument → cached config → Backstage `catalog-info.yaml` annotation → repo-name auto-detection → user prompt) and returns:

- Active incidents on the mapped service
- 90-day incident history summary
- File-level and structural risk signals in the diff
- Correlation findings between current changes and past incidents
- A risk score (0 = no risk, 5 = critical) with recommendations

## Signing in

The plugin configures one MCP server, `pagerduty`. Once your admin has completed the [one-time admin step](#before-you-start-one-time-admin-step), start Copilot CLI and sign in to PagerDuty when prompted (or connect `pagerduty` from `/mcp`). No API token, client ID, or client secret is needed.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Sign-in fails after login with an error that the app is not installed in the account | A PagerDuty account admin must [install the GitHub Copilot CLI app](https://app.pagerduty.com/oauth-apps/PEMCLUK) once, then sign in again. |
| Copilot CLI asks for a client ID | Enter `https://github.com/copilot/cli/client-metadata.json`. |
| `401` / unauthorized responses | Your session expired or was revoked. Re-authenticate the `pagerduty` MCP server. |
| `429` rate limit | The PagerDuty API is rate limiting requests. Wait a minute and retry. |

### Restricting scopes

The GitHub Copilot CLI app requests all scopes. If your organization wants narrower scopes, register your own Scoped OAuth app and connect with it as described in [Can't get the app installed?](#cant-get-the-app-installed).

## Reporting issues

File bugs and feature requests at https://github.com/PagerDuty/github-agenthq-plugins/issues. Security concerns go to **security@pagerduty.com** — see [SECURITY.md](SECURITY.md).

## License

Apache 2.0 — see [LICENSE](LICENSE).
