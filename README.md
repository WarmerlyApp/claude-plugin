<p align="center">
  <img src=".github/warmerly-icon.png" alt="Warmerly" width="96" height="96">
</p>

<h1 align="center">Warmerly for Claude</h1>

<p align="center">
  Cold email outreach, deliverability and reply triage from inside Claude.<br>
  <a href="https://warmerly.com/ai">Website</a> ·
  <a href="https://docs.warmerly.com/ai">Docs</a> ·
  <a href="https://github.com/WarmerlyApp/mcp">Other AI clients</a>
</p>

<p align="center">
  <img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-a34f1d">
  <img alt="MCP" src="https://img.shields.io/badge/MCP-streamable%20HTTP-a34f1d">
  <img alt="Claude Code plugin" src="https://img.shields.io/badge/Claude%20Code-plugin-a34f1d">
</p>

The official [Warmerly](https://warmerly.com) plugin for Claude. It connects Claude to your
Warmerly workspace over MCP and adds four workflow skills for cold email outreach.

| Skill | Use it to |
| --- | --- |
| `launch-outreach-campaign` | go from "who do I sell to" to a verified lead list, a sequence that follows cold-email best practice, a readiness check and a launch |
| `deliverability-doctor` | work out why mail lands in spam or bounces: mailbox health, SPF/DKIM/DMARC/MX, blocklists, placement tests, and the fix |
| `reply-triage` | sort real replies from autoresponders and unsubscribes, draft answers, and suppress people who asked to be left alone |
| `transactional-email-setup` | verify a sending domain for app email (password resets, receipts) and check delivery |

The MCP server is `https://app.warmerly.com/api/mcp`. You sign in with your normal Warmerly login
and pick one workspace; the connection only ever sees that workspace. Every Warmerly plan can
connect, including Free.

Full docs: https://docs.warmerly.com/ai

## Install in Claude Code

```text
/plugin marketplace add WarmerlyApp/claude-plugin
/plugin install warmerly@warmerly
```

Then run `/mcp`, pick `warmerly`, and sign in. The skills load automatically when a task matches
them, namespaced as `warmerly:launch-outreach-campaign` and so on.

## Use in claude.ai

1. **Connector:** Settings → Connectors → Add custom connector, URL
   `https://app.warmerly.com/api/mcp`, then sign in with your Warmerly account.
2. **Skills:** run `node build-skill-zips.mjs` (Node 18+, no dependencies). It writes one `.zip`
   per skill to `dist/`. Upload each under Settings → Capabilities → Skills.

## Example prompts

- "Find UK marketing agencies with 10–50 staff and launch a two-step campaign to them."
- "Why is my mailbox landing in spam? Check everything and tell me what to fix."
- "Go through this week's replies, draft answers to the real leads, and suppress anyone who asked
  to stop."
- "Set up transactional email for app.example.com and tell me which DNS records to add."

## What the skills will and will not do

- They act on real email. Unless you tell Claude it may act on its own, it asks before bulk
  adding leads, launching or resuming a campaign, or sending a reply.
- They never reply to an unsubscribe or a complaint; they suppress the address instead.
- Plan limits, the suppression list and Warmerly's sending protections apply to everything the
  assistant does, exactly as in the Warmerly dashboard. Every action is recorded in your
  workspace's audit log.
- Mailbox passwords never pass through Claude. Connecting a mailbox happens in the Warmerly
  dashboard.
- You can revoke the connection any time under Settings → Connected apps in Warmerly.

## Privacy

Warmerly's privacy policy: https://warmerly.com/privacy. The plugin itself stores nothing; the MCP
server acts only in the one workspace you authorise, and every write is recorded in that
workspace's audit log.

## Layout

```text
.claude-plugin/plugin.json       plugin manifest
.claude-plugin/marketplace.json  single-plugin marketplace (source "./")
.mcp.json                        the Warmerly MCP server
assets/icon.svg                  plugin icon
skills/<name>/SKILL.md           one workflow per skill
build-skill-zips.mjs             zips each skill for claude.ai (no dependencies)
```

## Support

hello@warmerly.com · [Privacy](https://warmerly.com/privacy) · [Terms](https://warmerly.com/terms)

## License

MIT. See [LICENSE](LICENSE).
