---
name: transactional-email-setup
description: Use when the user wants to send application email (password resets, receipts, notifications, sign-up confirmations) through Warmerly's transactional email API, verify a sending domain for it, check why a transactional email did not arrive, or send one through the Warmerly MCP server.
---

# Transactional email setup

Warmerly's transactional email is for mail a person expects because they did something: a
password reset, a receipt, an alert. It runs on separate mail servers from cold outreach and
warmup so that one can never damage the other. Keep it that way.

## Guardrails (read first)

- **Never send cold or marketing email from a transactional domain.** Not once, not "just a
  small list". Transactional mail rides on its own IP reputation; a single cold blast can push
  every password reset the user's product sends into spam. Cold outreach belongs in a campaign,
  from a separate domain (see the `launch-outreach-campaign` skill).
- **You send real email.** `send_transactional_email` delivers to a real inbox. Unless the user
  has explicitly given you autonomy, confirm the recipient and content before sending, and never
  loop it over a list of addresses.
- **Respect plan limits and the suppression list.** Suppressed recipients are dropped
  automatically; do not try to work around that by changing the address.

## Step 1: What exists

`list_sending_domains`: each domain with its status. `active` means verified and ready to send
from. `pending_dns` means records are not published yet. No domains means start at Step 2. If the
user's workspace does not have transactional email enabled yet, the dashboard shows "early access".

## Step 2: Add and verify a domain

Adding a domain happens in the dashboard at https://app.warmerly.com/transactional (Domains tab).
Recommend a subdomain dedicated to this job, such as `notify.example.com` or `mail.example.com`,
rather than the root domain, and never a domain the user already uses for cold outreach or for
Warmerly OneMail mailboxes (Warmerly refuses a domain that already has another job).

Warmerly shows three records to publish at the user's DNS provider:

1. **SPF:** an `include:` for Warmerly's transactional servers. If the domain already has an SPF
   record, add the include to that one record; do not publish a second SPF record.
2. **DKIM:** a TXT record with the key Warmerly generated. Copy it exactly, as one string.
3. **DMARC:** a `_dmarc` TXT record. Any valid DMARC record passes verification; recommend
   including a `rua=` report address so failures are visible.

Then press Verify in the dashboard. All three present makes the domain `active`. Use
`check_domain` on the domain to confirm what the world sees if verification keeps failing. DNS
changes can take from minutes to a few hours to appear.

**MX advice.** Warmerly does not add an MX record for a transactional domain: the user keeps
receiving mail wherever they do today. But a domain that cannot receive mail at all looks like a
throwaway spam domain to inbox providers. If `check_domain` shows no MX for the sending domain or
its parent, recommend adding one (the user's normal mail provider's MX, so replies reach a real inbox).

## Step 3: Send

In the user's application, the send goes through the transactional REST API with a transactional
API key (created in the dashboard's API keys tab), and every send should carry an
`Idempotency-Key` header unique to that event (for example `password-reset:<userId>:<token id>`).
A retry with the same key never sends twice: it returns the first message and marks it a duplicate.
`search_docs` for "transactional" finds the full API reference.

From here, `send_transactional_email` sends one message directly, which is useful for a test or a
one-off. The From address must be on an `active` domain. A good test email:

- has a real subject like "Your sending domain notify.example.com is connected", not "test";
- has both a text and an HTML part and a line saying why it was sent;
- goes to an address the user controls.

The tool makes a retry of an identical send safe on its own. If it reports that a send may have
been interrupted, do not send again blindly: look it up first (Step 4).

If `send_transactional_email` is not available on this connection, have the user send a test from
the dashboard at https://app.warmerly.com/transactional.

## Step 4: Did it arrive?

`list_transactional_emails` (filter by `status`, for example `bounced` or `failed`) and
`get_transactional_email` for the event timeline of one message:

- `sent`: accepted by Warmerly's server. Not yet proof of delivery.
- `delivered`: the recipient's server accepted it. If the user still cannot find it, it is in
  spam or a filter: check the domain with `check_domain` and the content (one-line "test" emails
  get filtered).
- `deferred`: the recipient's server said try later; Warmerly retries.
- `bounced`: permanent failure; the reason is in the timeline. The address is suppressed afterwards.
- `suppressed`: the recipient was on the suppression list, so nothing was sent.
- `failed`: the submission failed. The timeline says whether it may have gone out.

## When a tool is missing

`send_transactional_email` may not be available yet. Domain setup and API keys are always done in
the dashboard at https://app.warmerly.com/transactional; nothing in this workflow needs a password
or key to pass through you, and you should never ask the user to paste an API key into the chat.
