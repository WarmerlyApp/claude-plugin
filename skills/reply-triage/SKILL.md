---
name: reply-triage
description: Use when the user wants to go through their Warmerly inbox, find replies that need an answer, sort interested leads from autoresponders and unsubscribes, or draft and send replies to campaign responses. Requires the Warmerly MCP server.
---

# Reply triage

Go through the conversations that are waiting on the user, sort them honestly, and answer the
ones worth answering. Speed matters for a warm reply; accuracy matters more. One wrong reply to
someone who asked to be left alone does more harm than ten slow ones.

## Guardrails (read first)

- **You are writing to real people as the user.** Every reply goes out from their mailbox under
  their name. Unless the user has explicitly told you to send on your own, **show each reply and
  get a yes before `send_reply`.** Drafting is always fine; sending is the step that needs consent.
- **Never auto-reply to an unsubscribe request, a complaint, an angry or threatening message, or
  a legal notice.** Suppress the address instead (below) and tell the user. A reply to "stop
  emailing me" is itself the next complaint.
- **Respect plan limits.** AI drafting spends AI credits. If a tool refuses on a limit, say so and
  point to https://app.warmerly.com/settings/usage.
- **Never invent facts** in a reply: prices, availability, case studies, meeting times. If the
  answer needs something you do not know, ask the user or leave a clearly marked gap.

## Step 1: What is waiting

`list_conversations` with `filter` set to `needs_reply` (the newest message is inbound and the
contact is a lead). Use `unread` for "anything new", and `q` to search for one person or company.
Up to 50 per call; say how many there are in total.

## Step 2: Read before judging

`get_conversation` for each one. It returns the timeline (latest 50 messages), any saved draft,
and which mailbox a reply should send from. Reading through the tools does not mark anything as
read for the user.

## Step 3: Sort each one

| Kind | How to spot it | What to do |
| --- | --- | --- |
| **Interested** | asks a question, wants a call, asks for pricing or details | Draft a reply. Top of the list for the user. |
| **Referral** | "talk to Jane, she handles this" | Draft a short thank-you; suggest adding Jane as a lead (do not email her without the user's yes). |
| **Not now** | "maybe next quarter", "not a priority" | Short, gracious reply; note the timing for the user. |
| **Not interested** | a polite no | Usually no reply needed, or one line of thanks. Categorize `not_relevant`. |
| **Unsubscribe / angry / complaint** | "remove me", "stop", "unsubscribe", "how did you get my address", anger, legal threats | **Do not reply.** `suppress_emails` with their address, categorize `not_relevant`, tell the user. |
| **Autoresponder** | out of office, "I'm away until", ticket numbers, "this mailbox is not monitored", "no longer with the company" | **Not a reply.** Do not answer and do not count it as a response. If it names a replacement contact, mention it. If the person has left, suggest suppressing the old address. |
| **Bounce / delivery failure** | mailer-daemon, "delivery failed", "address not found" | Not a reply. Suppress the address. |
| **Spam** | unrelated selling, phishing | Categorize `spam`. |

Most campaign "replies" are autoresponders. Say so when you report numbers, so the user does not
read an out-of-office wave as interest.

Use `categorize_message` with `lead`, `not_relevant` or `spam` to file each conversation, so the
user's inbox matches your sort.

## Step 4: Draft

`draft_reply` generates a draft from the conversation. Then edit it to these rules before showing
the user:

- Answer the question they asked, first. Two to five sentences.
- Plain text, no em dashes, every sentence starting with a capital letter.
- One clear next step (a time to talk, a link, one question).
- Match their tone; if they wrote two lines, do not send six.
- Nothing that was not in the conversation or given to you by the user.

## Step 5: Send

With the user's yes (or explicit autonomy), `send_reply` with the conversation's contact and the
final body. Send from the mailbox `get_conversation` named unless the user says otherwise; it keeps
the thread on the address the lead already knows.

## Step 6: Report

Finish with a short summary: how many conversations, how many were real replies, who is
interested (name, company, what they want), what you sent, what you suppressed, and what needs
the user's own answer.

## When a tool is missing

`draft_reply`, `send_reply`, `categorize_message` and `suppress_emails` may not be available on
the user's connection yet. If one is not in your tool list, do not pretend to call it. Write the
draft in the chat for the user to paste, and send them to https://app.warmerly.com/inbox to send
it and file the conversation, and to https://app.warmerly.com/settings/suppression to add
addresses to the suppression list.
