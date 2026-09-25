---
name: deliverability-doctor
description: Use when the user's Warmerly emails are landing in spam, bouncing, getting few replies, when a mailbox's health score dropped, or when they ask to check SPF, DKIM, DMARC, MX or blocklists for a domain. Diagnoses mailboxes and domains through the Warmerly MCP server and gives concrete DNS and sending fixes.
---

# Deliverability doctor

Diagnose why mail is not reaching the inbox, then give fixes the user can actually apply. Work
from evidence (health history, DNS checks, placement tests), never from a hunch, and never tell
someone their domain is fine because a check could not run.

## Guardrails (read first)

- **You act on real mailboxes.** Diagnosis is read-only and safe. Anything that changes sending
  (turning warmup off or on with `set_warmup`, pausing a campaign, running a placement test that
  spends a credit) needs the user's yes unless they have explicitly given you autonomy.
- **Respect plan limits.** Placement tests are metered. Check `get_account_overview` for quota
  before running one, and do not run several "just to see".
- **You cannot edit DNS.** You can tell the user the exact record to publish. They, or whoever
  runs their DNS, make the change.

## Step 1: Find the patient

1. `list_mailboxes`: status, warmup on or off, today's volume and the latest health score for each.
   Paused, disconnected or needs-reconnect mailboxes are the first suspects.
2. If the user named a campaign rather than a mailbox, `get_campaign` shows its settings and
   readiness; check every mailbox it sends from.

## Step 2: History, not a snapshot

`get_mailbox_health` for each suspect mailbox (14 days by default, up to 90). Look for:

- **Bounce rate above 3%:** the list is the problem, not the DNS. Fix the lead source and verify
  addresses before the next send (`verify_email`, or https://app.warmerly.com/verify).
- **Inbox rate falling over several days** while volume rose: the mailbox is sending more than its
  reputation supports. Lower the campaign's daily volume and keep warmup on.
- **Sudden drop on one day:** look at what changed that day (a new campaign, a DNS change, a
  new domain in links).
- **Reply rate near zero with a good inbox rate:** that is copy or targeting, not deliverability.
  Say so plainly.

The same call returns the latest DNS check for that mailbox.

## Step 3: Check the domain live

`check_domain` with the bare sending domain (the part after the @, for example `acme.com`,
no `https://`). Read each verdict:

| Record | Good | Fix when it is not |
| --- | --- | --- |
| SPF | one `v=spf1` record that includes the provider's servers and ends in `-all` or `~all` | Merge into ONE record (two SPF records is a failure). Stay under 10 DNS lookups: drop includes for services that no longer send. Never use `+all`. An IPv6 address after `ip4:` voids the whole record. |
| DKIM | a key of 2048 bits, not in test mode (`t=y`) | Turn DKIM signing on at the mail provider and publish the key it gives you. Replace 1024-bit keys. |
| DMARC | `v=DMARC1` with a `rua=` address for reports | Start at `p=none` with `rua=` so reports arrive, then move to `p=quarantine` once SPF and DKIM pass for all legitimate senders. |
| MX | at least one MX record | A sending domain that cannot receive mail looks like a throwaway spam domain, and warmup needs replies. Add the provider's MX records. |

How to read the result honestly:

- **"unknown" is not a pass.** It means the lookup could not run (a timeout, a resolver refusal).
  Say "could not be checked" and try again later; never report it as fine.
- **Advisories are not failures.** `p=none` without a report address is common and worth fixing,
  but it does not explain a sudden spam problem on its own. Lead with hard failures.
- **Gmail, Outlook and other shared consumer domains:** the records belong to the provider, and
  Gmail signs with a selector that cannot be discovered, so DKIM can read as missing when it is
  not. Tell the user there is nothing to set up on those domains, and that cold outreach from a
  free address is itself the deliverability problem: they should send from their own domain.

## Step 4: Blocklists

The mailbox's page at `https://app.warmerly.com/accounts/<mailboxId>` shows the domain against
the major domain blocklists (Spamhaus DBL, SURBL, URIBL and others) and the mail server's IP
against IP lists (Spamhaus ZEN, SpamCop, Barracuda and others). When reading them to the user:

- A "policy" listing such as Spamhaus PBL covers whole consumer IP ranges on principle. It is not
  an abuse report and is not something to fix.
- For mailboxes on big providers (Google, Microsoft and similar), Warmerly does not check the
  mail server's IP at all: the server they submit to is not the address mail leaves from, and the
  customer could not get it delisted anyway. No IP result there is expected, not a gap.
- A real domain listing: stop campaign sending from that domain, find the cause (a bought list,
  a complaint spike), then request removal at the list's own delisting page. Some lists (for
  example UCEPROTECT) charge for delisting; do not recommend paying.

## Step 5: Prove it with a placement test

When DNS is clean and the user still reports spam placement, `run_placement_test` for the
mailbox (with their go-ahead: it is metered). It sends to seed inboxes and reports where the
message landed per provider. If the tool is not available, send the user to the mailbox page
at `https://app.warmerly.com/accounts/<mailboxId>` to run one.

## Step 6: Prescribe

Give a short ordered list, most important first, each with the exact change. Typical order:

1. Fix hard DNS failures (SPF, DKIM, missing MX).
2. Stop the bleeding: `pause_campaign` on a campaign bouncing above 3% (with the user's yes).
3. Keep warmup on (`set_warmup`) for any mailbox that is sending campaigns. Never turn warmup
   off on a mailbox that is having trouble.
4. Reduce volume per mailbox; spread sending across more mailboxes instead.
5. Turn off open and click tracking on cold campaigns unless a verified tracking domain is set.
6. Clean the list: verify addresses, remove role addresses that bounce.
7. Add a DMARC report address and review reports weekly.

## When a tool is missing

`set_warmup`, `run_placement_test`, `pause_campaign` and `verify_email` may not be available on
the user's connection yet. If one is not in your tool list, do not pretend to call it. Tell the
user where to do it: mailboxes and warmup at https://app.warmerly.com/accounts, campaigns at
https://app.warmerly.com/campaigns, verification at https://app.warmerly.com/verify.
`search_docs` finds the help pages for DNS setup by provider.
