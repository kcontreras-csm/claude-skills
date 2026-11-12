---
name: Case SP Triage
version: 1.0.0
description: Review all of the current user's open Salesforce support cases and give a neutral, evidence-based recommendation for which ones should move to Solution Provided, which need another follow-up touch first, which should stay where they are, and which are in the wrong Status or Sub-Status (e.g. Need More Information / Waiting on Customer while we owe the next step, or Solution Provided after the customer replied with an open issue). Read-only. Use when the user says "SP triage", "which cases to move to solution provided", "review my cases", "status cleanup", "clean my queue" or "metrics cleanup" (e.g. /case-sp-triage).
---

Review every open case owned by the current user and recommend, neutrally and with evidence, which status each one should be in. The goal is accurate metrics: cases that are really waiting on a confirmation should not sit in Need More Information, and cases where we still owe work should not be parked in Solution Provided.

Read-only: never change a case status, send an email, or post a comment. The user makes every move.

## Follow-up rules (edit here when the team's rules change)

*Default adopted 2026-11-11. Replace with the team's official rules when available.*

- **3-touch cadence:** touch 1 = our solution or question, touch 2 = reminder after **2 business days** with no reply, touch 3 = final notice after **2 more business days** with no reply.
- **Business days:** Monday to Friday, in the user's time zone (from `getUserInfo`, e.g. America/Mexico_City). Count whole business days since the last outbound message.
- **Reply = substantive reply.** Emoji reactions ("reacted to your message"), auto-replies, out-of-office messages and bounces are not replies, but note them (an out-of-office reply with a return date is a good reason to pause the cadence).

## Status fields (OrgCS)

Every case has two status fields, and both must match the conversation:

- `Status`: e.g. Working, Need More Information, Solution Provided, Call/Meeting Scheduled.
- `Sub_Status__c`: e.g. In Progress, Waiting on Customer, Waiting on Internal, Solution Provided, Manually Scheduled.
- `Unresponsive_Follow_ups__c`: a counter of our follow-ups without a customer reply. Use it as a cross-check on your own touch count. If the counter is high but the customer says they already answered, we probably missed their reply (see the Common-sense checks).

Metrics pollution usually shows up as **Need More Information / Waiting on Customer while we owe the next step**. That makes the case look customer-pending when it's really on us. Flag it every time.

## Decision rules

Apply in this order. The first rule that matches wins.

1. **Keep: ball in our court.** The customer's latest substantive message is newer than our latest outbound one, or we promised something we haven't delivered (a callback, a check with a team, a swarm or backend action). Never recommend SP here; that would hide pending work.
2. **Keep: waiting on an internal dependency** (swarm, another team, bug, engineering, "we're reviewing internally"). The right status is Working / Waiting on Internal. Never SP.
3. **Keep: call or meeting scheduled** and the meeting hasn't happened yet (check the emails for the date). After the meeting, re-evaluate with the rules below.
4. **Move to SP: customer confirmed.** The customer says it worked, it's resolved, or asks to close. Suggest SP or closing per team practice, and /case-closure for the confirmation email.
5. **Move to SP: solution delivered.** Our last substantive outbound message gave a complete answer, fix or workaround, or confirmed that the requested action was done, and nothing new is open from the customer. This applies right away, without waiting for touches. If our message also asked a blocking question that the fix depends on, it is not a delivered solution; use rule 6.
6. **Need More Information, cadence running.** We asked the customer for something blocking and they haven't replied:
   - Fewer than 2 business days since our last touch: **Keep (NMI)**.
   - Touch 2 or 3 is due: **Follow-up due**. Say which touch and suggest /case-reminder (touch 2) or /case-closure no-response (touch 3).
   - All 3 touches sent and 2 more business days with no reply: **Move to SP / closure candidate**.
7. **Already in SP:**
   - The customer replied after the move with an open issue or a new question: **Move back out of SP** (to NMI or in progress), because the case is no longer pending confirmation.
   - No reply and the cadence ran out: **Closure candidate** (suggest /case-closure).
   - Otherwise: **Keep (SP)**.
8. **Status mismatch (applies on top of rules 1-2).** If rule 1 or 2 matched but the case is in Need More Information / Waiting on Customer (or SP), don't put it in "Keep". Put it in **Status mismatch: we owe the next step**, name the right status (usually Working / In Progress, or Working / Waiting on Internal), and say what we owe the customer. Also flag the reverse: Working or Waiting on Internal while the next step is clearly the customer's.
9. **Needs your review.** Anything ambiguous: mixed signals, conflicting messages, data you couldn't read, or the customer's reply is unclear. Don't force a call.

## Common-sense checks (apply on top of the rules)

- **Neutral, not aggressive.** Don't recommend SP just because a case is old or quiet. Only recommend it when the evidence says the next step is the customer's confirmation. Equally, don't keep cases open out of caution when a solution was clearly given.
- **The status must match reality.** NMI means we are waiting for information. SP means we gave the solution and are waiting for confirmation. Flag mismatches both ways.
- **Partial answers:** if we solved one of several asks, it's not SP. Name what's still open.
- **Escalated, high priority or VIP-looking cases** (IsEscalated = true, low Priority number or an escalation note): never recommend closure; at most SP, and flag them for review.
- **Fresh cases:** a case created today, or one whose first contact went out today, is almost never SP-ready unless the request was a one-shot action that's already done.
- **The customer's tone counts:** frustration or "still not working" overrides any timer.
- **"I already answered" signals:** if the customer says they already replied ("I have responded twice", "as I said before"), or `Unresponsive_Follow_ups__c` is high while the customer is engaged, assume we missed a reply. Their reply may be in a comment, from a different address or domain, or in a separate thread. Put the case at the top of Status mismatch and tell the user to check the full history before answering.
- **A different sender:** a reply from someone other than the case contact (a colleague or a shared inbox) is still a customer reply, but say so, especially if it's a new request.
- **A closing notice already sent:** if we already sent a closing email (no-response or confirmation) and the only thing since is a new out-of-scope request that we redirected, recommend closing rather than SP.
- **Time-limited access:** if the customer granted login access for N days and we still owe the investigation, mention the expiry date. It's a reason to act now.
- **New scope in a reply:** if the customer adds a new request (e.g. right before a call), note it so it gets covered. It doesn't change the status by itself.
- **Don't guess.** If the email bodies can't be read, say so and classify as "Needs your review".

## Steps

1. **Tools:** load via ToolSearch `select:mcp__orgcs__soqlQuery,mcp__orgcs__getUserInfo`. Get the user Id, time zone and today's date from `getUserInfo`.
2. **List open cases:**
   `SELECT Id, CaseNumber, Subject, Status, Sub_Status__c, Unresponsive_Follow_ups__c, Priority, IsEscalated, CreatedDate, LastModifiedDate, Contact.Name, Account.Name, (SELECT Incoming, MessageDate, Subject FROM EmailMessages ORDER BY MessageDate DESC LIMIT 6) FROM Case WHERE OwnerId = '<userId>' AND IsClosed = false ORDER BY LastModifiedDate ASC LIMIT 100`
   If the arguments name specific statuses or case numbers, filter to those.
3. **Read the conversation, comments first** (they're short and usually tell most of the story):
   - **All comments in one query:** `SELECT Parent.CaseNumber, CommentBody, IsPublished, CreatedBy.Name, CreatedDate FROM CaseComment WHERE ParentId IN (<all case Ids>) AND CreatedDate = LAST_N_DAYS:30 ORDER BY CreatedDate DESC LIMIT 100`. Published comments starting with `<Created By: <contact name>>` are customer replies. Our outbound emails are usually mirrored as published comments, often prefixed with the status that was set (e.g. `[Need More Information]`). Internal (unpublished) comments can reveal internal dependencies (swarm, transfers, bugs).
   - **Emails only where needed:** from the subquery in step 2, find the cases where the latest email is incoming, or where the comments don't show the content. Fetch those by Id in one query: `SELECT Parent.CaseNumber, Incoming, MessageDate, FromAddress, TextBody FROM EmailMessage WHERE Id IN (<email Ids>)`.
   - **Large results:** email bodies include the full quoted history and are often saved to a file. Trim them with a small `python3 -I` script that, for each record, keeps only the text before the first `From:`, `-----Original Message` or `On ... wrote:` line, and prints at most ~600 characters (~350 for comments). Never print full bodies in the output.
   - **Status history (optional):** `SELECT CaseId, Field, OldValue, NewValue, CreatedDate FROM CaseHistory WHERE CaseId IN (<Ids>) AND Field IN ('Status','Sub_Status__c') AND CreatedDate = LAST_N_DAYS:14 ORDER BY CreatedDate DESC LIMIT 50`. Use it to see when a case entered SP, or whether it was flipped from Waiting on Internal to Waiting on Customer by a holding message.
   - Run up to 3 queries in parallel at a time; bigger batches have timed out.
   - If SOQL is blocked by the security policy, follow the fallback in the case-first-contact skill (SOSL, then `Task WHERE WhatId = '<Id>'` for logged emails), and mark what couldn't be read.
4. **For each case, work out:** the current Status / Sub_Status, the last outbound date, the last substantive inbound date, who has the ball, what our last outbound message did (solution, question, reminder, final notice, holding message), how many touches were sent since the last customer reply, and business days elapsed.
5. **Apply the Decision rules and the Common-sense checks.** Give one recommendation per case.
6. **Final validation: judgment pass on the conversation.** Before writing the output, re-read the latest exchange of each case that isn't in "Keep as is" (and any "Keep" case whose call felt borderline). Step back from the rules and ask, as an experienced Success Engineer would:
   - **Did we really answer what they asked?** Compare the customer's original request and their latest message with our last reply. A reply on a related topic, a generic article link, or "please try X" without confirming that X applies is not a delivered solution.
   - **Would the customer be surprised or annoyed** to see the case in this status? If a reasonable customer would think "but I'm still waiting on them", don't move it to SP.
   - **Are there signals the rules missed?** For example: "I'll test it tomorrow" or "let me check with my team" (the customer is engaged, so give them time and treat it like SP-pending rather than a missed touch), a reply sent from a different person or address, a promised date, a thread split across several emails, or a holiday or out-of-office message.
   - **Is the rule outcome too mechanical?** The timer says close, but the last message was a complex technical ask sent late on a Friday. Or the timer says wait, but the customer clearly said "thanks, all good".
   - **Has anything changed** since the data was pulled? Run one check: `SELECT Parent.CaseNumber, Incoming, MessageDate FROM EmailMessage WHERE ParentId IN (<Ids>) AND MessageDate > <time of the first pull>`. A new email or comment from the last few minutes overrides the earlier call.
   - **Does the status match who has the ball?** Check `Status` and `Sub_Status__c` against your conclusion, and move any mismatch to Status mismatch.

   If the judgment pass disagrees with the rule outcome, change the recommendation and add **"Adjusted on review:"** with a one-line reason to that case's Why line. If it confirms the outcome, don't mention it. If you're still unsure, move the case to "Needs your review" instead of guessing. Stay neutral: this pass may move cases toward SP or away from it.

## Output

Start with a one-line count, e.g. "14 open cases reviewed: 3 move to SP, 2 ready to close, 3 status mismatches, 1 follow-up due, 5 keep."

Then one section per bucket, in this order, skipping empty buckets:

1. **Move to SP**
2. **Move back out of SP**
3. **Closure candidates**
4. **Status mismatch: we owe the next step** (most urgent first, e.g. a customer who says we missed their reply)
5. **Follow-up due** (say which touch, and the skill to run)
6. **Needs your review**
7. **Keep as is** (compact: one line per case)

For each case in buckets 1-6, show:
- [CaseNumber](https://orgcs.lightning.force.com/lightning/r/Case/<Id>/view): Subject | Contact | Current Status / Sub_Status
- **Why:** one sentence based on evidence, with dates (e.g. "Solution sent Nov 6 (Welcome email resent); no reply in 3 business days, no open question").
- **Next:** the exact action (e.g. "Change status to Solution Provided", "Run /case-reminder 481546772").

If the final validation changed any recommendation, add one line after the count, e.g. "2 recommendations adjusted on review (<case number>, <case number>)."

End with an **Assumptions** line if any (cadence used, unreadable data, out-of-office pauses). Keep it scannable: no long paragraphs, no email bodies, no confidence scores.
