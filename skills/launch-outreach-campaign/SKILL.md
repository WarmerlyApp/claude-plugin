---
name: launch-outreach-campaign
description: Use when the user wants to start, plan or launch a cold email outreach campaign in Warmerly, find leads for one, write the email sequence, or check whether a campaign is ready to send. Walks from ideal customer profile to lead search, verification, campaign creation, sequence writing, readiness and launch, then monitoring. Requires the Warmerly MCP server.
---

# Launch an outreach campaign in Warmerly

You are helping someone send cold email to real people from their real mailboxes. A bad
campaign burns their domain's reputation for weeks and can get their mailboxes blocked. Go in
order, and do not skip the readiness steps to get to "launched" faster.

## Guardrails (read first)

- **You act on real email.** Every lead you add can receive mail, and every launch sends it.
  Unless the user has explicitly said you may act on your own, **show them the plan and get a yes
  before adding leads in bulk, launching, or resuming a campaign.** A clear "go ahead and launch it"
  counts. "Help me with a campaign" does not.
- **Respect plan limits.** Call `get_account_overview` first. It shows the plan and quota usage.
  If a step would exceed a limit, say so and point the user to https://app.warmerly.com/settings/usage
  rather than looking for a way around it. Tools refuse over-limit actions anyway; say why when they do.
- **Never invent data.** Only use leads the lead search returned or the user gave you. Do not guess
  email addresses. Do not make up facts about a company for personalization.
- **Rate limits are real.** 60 tool calls a minute per connection, 20 a minute for tools that send.
  Batch work; do not loop one lead at a time.

## Step 1: Is there anything to send from?

1. `get_account_overview`: note the plan, quotas and any setup gaps.
2. `list_mailboxes`: you need at least one connected mailbox whose status is healthy, with warmup on.
   - **No mailboxes:** call `get_connect_mailbox_link` and give the user the URL. Mailbox
     passwords and OAuth never pass through you; they connect it themselves.
   - **Warmup off, or a mailbox connected in the last two weeks:** recommend warming for at least
     14 days (21 is better) before any campaign sends from it. A cold new mailbox sending cold email
     is the fastest way to the spam folder.
   - **Low health score or a failing DNS check:** stop and use the `deliverability-doctor` skill
     first. Fixing SPF/DKIM/DMARC before launch is far cheaper than after.

## Step 2: Pin down the ideal customer profile

Ask, if the user has not said: who they sell to (industry, country, city, company size), what
they offer, and the one outcome they want from a reply (a call, a demo, a quote). Keep the
answer as one sentence you can search with, for example "dental clinics in Manchester with 11 to 50 staff".

## Step 3: Find leads

1. `count_leads` with that sentence as `query`. Tell the user the size of the segment. If the
   result says it searched as plain keywords, the AI interpretation failed. Say so and offer to
   narrow with structured `filters` (country as an ISO code such as `GB`, city, category, employee band).
2. `search_leads` with a small `limit` (10) and show a sample: company, domain, city, and whether
   it has a contact email (`hasEmail`). Addresses themselves are not returned by a search; they
   reach a campaign through `add_leads_from_search`, which spends the lead-export quota.
   Ask whether the sample looks right before going further. Adjust filters until it does.
3. Prefer segments where most companies have `hasEmail: true`.

## Step 4: Verify before you add

Bounces above about 3% get mailboxes filtered. Leads pulled with `add_leads_from_search` are
checked by Warmerly's import gate as they are added. For a list the user supplied, if
`verify_email` is available, verify the addresses before adding them, or at least a sample of 20
to judge the list's quality. Do not add addresses that come back invalid. If verification is not
available through the tools, tell the user the leads will be checked by Warmerly's send-time gate,
and that they can bulk verify at https://app.warmerly.com/verify.

## Step 5: Create the campaign and add leads

- `create_campaign` with a descriptive name (segment plus offer, for example "Manchester dentists, Q4 booking tool").
- Add leads with `add_leads_from_search` (same query or filters you tested) or `add_leads` for a
  list the user supplied. Confirm the count with the user first unless they gave you autonomy.
  Report how many were added and skipped, and why.

If these tools are not available, tell the user to create the campaign at
https://app.warmerly.com/campaigns and push the leads from https://app.warmerly.com/leads using
"Add to campaign", then carry on from Step 6 using `get_campaign` once they have.

## Step 6: Write the sequence

Write it with the user, then save it with `set_campaign_sequence` (a simple list of steps, each
with a subject, a body and the days to wait before it). If that tool is not available, give the
user the finished copy to paste into the campaign's Sequence tab.

Copy rules. These come from what actually got replies, so hold to them:

- **Short and plain.** The first email is 50 to 90 words, plain text, no images, no attachments,
  at most one link. It should read like one person writing to another.
- **One call to action,** and make it a small ask ("Worth a quick look?"), not "book a 30 minute demo".
- **No em dashes** anywhere in subject or body. Use a full stop or a comma instead.
- **No lowercase word after a full stop.** Every sentence starts with a capital, including after
  a merge tag.
- **Subject lines:** two to five words, lowercase is fine, no clickbait, no "Re:" or "Fwd:" on a first email.
- **Merge tags:** only `{{firstName}}`, `{{lastName}}`, `{{companyName}}`, `{{email}}`,
  `{{senderFirstName}}`, `{{senderLastName}}`, `{{senderName}}`, `{{senderEmail}}`, `{{ai_opener}}`,
  or a custom variable the leads actually carry. A misspelt tag (`{{company}}`) renders as a blank
  for every recipient. Write sentences that still read well if `{{firstName}}` is empty, or
  open with "Hi there".
- **`{{ai_opener}}`** writes a personalised first line per lead. It needs a campaign brief, and it
  spends AI credits. Only use it if the user wants it.
- **Follow-ups:** two or three, 3 to 5 days apart, each shorter than the last, each adding one new
  reason to reply. The last one politely closes the loop.
- **No spam triggers:** avoid "free", "guarantee", "act now", all caps, several exclamation marks
  and money symbols in subject lines.

## Step 7: Readiness

`get_campaign` returns `readiness`: a list of checks, each `ok` or not, with a severity.

- A failed check with severity `error` blocks the launch. Fix it or tell the user exactly what to
  fix and where.
- A `warning` does not block, but read every one to the user in plain words. Common ones:
  a free mailbox still waiting out its warmup (the campaign starts sending when it unlocks),
  tracking on without a verified tracking domain (turn tracking off for cold email),
  content-quality flags, variables that do not exist on the leads.

Know the real sending ceiling before promising a date: each mailbox sends at most one email per
"minimum wait" gap inside the sending window, so four mailboxes pacing one email every 12 minutes
across 8 hours top out near 160 a day, whatever the daily limit says. Warmup shares those
mailboxes, so the real number is lower. More mailboxes, not a higher daily limit, is what raises it.

## Step 8: Launch

With the user's go-ahead (or explicit autonomy), call `launch_campaign`. If it returns blocking
issues nothing was sent; explain them. If it succeeds with warnings, repeat the warnings.
If `launch_campaign` is not available, send the user to the campaign page at
`https://app.warmerly.com/campaigns/<campaignId>` and have them press Start.

## Step 9: Monitor

After a day or two, `get_campaign_stats` (or `list_campaigns` for the counts) and read it like this:

- **Bounce rate above 3%:** pause with `pause_campaign`, and find out why before sending more. The
  lead source or the verification is the usual cause.
- **No sends in 48 hours** on an active campaign: check `get_campaign` readiness and mailbox health.
- **Reply rate:** do not judge until there are a few hundred sends. Most early "replies" are
  autoresponders; use the `reply-triage` skill to separate real ones.
- **Replies arrive:** hand over to `reply-triage`.

`resume_campaign` sends again, so it needs the same go-ahead as a launch.

## When a tool is missing

Some tools above (`create_campaign`, `set_campaign_sequence`, `add_leads`, `add_leads_from_search`,
`verify_email`, `launch_campaign`, `pause_campaign`, `resume_campaign`, `get_campaign_stats`) may not be
available on the user's connection yet. If a tool is not in your tool list, do not pretend to call
it. Do the parts you can, and give the user the exact dashboard page to finish the step:
campaigns at https://app.warmerly.com/campaigns, leads at https://app.warmerly.com/leads,
verification at https://app.warmerly.com/verify. `search_docs` can find the help page for any step.
