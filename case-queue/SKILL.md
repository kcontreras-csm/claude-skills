---
name: Case Queue
version: 1.0.0
description: Daily assistant for the current user's whole open case queue (Working, Need More Information, Solution Provided and any other open status). Reads what is on screen in Satellite (the case list view or the open case) and the OrgCS data, then says what to do today: SLA at risk, follow-ups due or overdue against the manager's cadence, status mismatches, and which skill to run per case (/case-reminder, /case-summary, /case-first-contact continue, /case-handover, /case-sp-triage). Read-only. Use when the user says "queue", "my queue", "mis casos", "que casos atiendo hoy", "como va mi cola", "prioridades de hoy", "revisa mis casos asignados", "tiempos de contacto", "a quien le toca seguimiento" (e.g. /case-queue, /case-queue working).
---

Credits: built by Abraham Zaid Aguilar with Claude on 2026-11-12. It builds on the work of Kevin Alexis Contreras: it calls his case skills (case-first-contact port, case-reminder, case-handover, case-summary, case-sp-triage) and borrows the status logic of case-sp-triage. The cadence is the manager rule of Oct 26 from the OrgCS Case First-Contact Prep skill by Abraham Zaid Aguilar.

Give the user a prioritized view of their open cases and tell them what to do next, case by case. Read-only: never change a status, send an email, post a comment or edit a case. The user makes every move. When a case needs a draft, call the matching skill (step 7) so the draft is ready; the user still reviews and sends it.

If the arguments name statuses or case numbers (e.g. `working`, `nmi`, `solution provided`, `476196596`), limit the review to those. With no arguments, review every open case the user owns.

## Cadence rules (manager rule, Oct 26; edit here when they change)

- **Working:** update the customer every 2 days (progress or a documented blocker).
- **Need More Information:** follow up every 3 days. At 7 days with no customer reply, the next message may tell the customer the case will close in 48 hours without a response. This is the only situation where announcing a closure is allowed.
- **Open Structured Investigation:** case note every 3-4 days confirming the SI is active and the customer is informed.
- **High priority** (IsEscalated, or Priority at the top of the scale): daily follow-up, no exceptions.
- **Solution Provided:** waiting for the customer's confirmation. Flag it as a closure candidate when the follow-ups after the solution ran out with no reply; never recommend closing an escalated or high-priority case, flag it for review.
- **More than 3 days without any activity** on a case is an alert: flag it at the top.
- Count days as business days (Mon-Fri) in the user's time zone from `getUserInfo`, since the last outbound message. State this as an assumption in the output.
- A reply is a substantive customer message. Emoji reactions ("reacted to your message"), auto-replies, out-of-office and bounces are not replies, but note them.
- `/case-sp-triage` uses its own default cadence (adopted 2026-11-11). If the two disagree, this rule set is the one to follow and the difference is worth mentioning once.

## Steps

1. **Tools:** load via ToolSearch `select:mcp__orgcs__soqlQuery,mcp__orgcs__getUserInfo`. Get the user Id, name, time zone and today's date from `getUserInfo`. Never reuse a name from an earlier run.
2. **Look at the screen (optional, best effort):** load `select:mcp__satellite__list_open_pages,mcp__satellite__read_active_page`. Satellite asks the user once for permission to share pages. If it is refused, unavailable, or the page is only an outline, skip this step and say so in one line.
   - If the page is a **case list view** (URL contains `/lightning/o/Case/list`), take the case numbers and the list name or filter shown (e.g. "sla") as the scope, and note the on-screen columns and badges (Response Target, Age, "late", "No summary") as extra signals. The screen is a hint; the data from step 3 decides.
   - If the page is a **case record** (`/lightning/r/Case/500.../view`), take that case Id and mark it "on screen". Still review the whole queue unless the arguments narrow it.
   - Anything else: ignore the screen.
3. **Pull the queue from OrgCS** (run up to 3 queries in parallel; bigger batches have timed out):
   - Cases: `SELECT Id, CaseNumber, Subject, Status, Sub_Status__c, Unresponsive_Follow_ups__c, Priority, IsEscalated, CreatedDate, LastModifiedDate, Contact.Name, Account.Name, (SELECT Incoming, MessageDate FROM EmailMessages ORDER BY MessageDate DESC LIMIT 6) FROM Case WHERE OwnerId = '<userId>' AND IsClosed = false ORDER BY LastModifiedDate ASC LIMIT 100`
   - SLA: `SELECT CaseId, MilestoneType.Name, TargetDate, IsCompleted, IsViolated FROM CaseMilestone WHERE CaseId IN (<Ids>) AND IsCompleted = false`. If it is not readable, use the Response Target from the screen and say so.
   - Latest customer activity: `SELECT Parent.CaseNumber, CommentBody, IsPublished, CreatedBy.Name, CreatedDate FROM CaseComment WHERE ParentId IN (<Ids>) AND CreatedDate = LAST_N_DAYS:14 ORDER BY CreatedDate DESC LIMIT 100`. Published comments starting with `<Created By: <contact name>>` are customer replies. Fetch email bodies only for cases whose latest email is incoming, and print at most ~300 characters of each.
   - If SOQL is blocked by the security policy, follow the fallback in the case-first-contact skill (SOSL, then `Task WHERE WhatId = '<Id>'`) and mark what could not be read.
4. **For each case work out:** Status / Sub_Status, last outbound date, last substantive customer message, who has the ball, business days since our last touch, the next contact due date under the cadence, and the SLA state (time to target, or violated).
5. **Classify and order.** The first match wins:
   1. **Act now:** SLA violated or due within 4 hours, a customer reply we have not answered, a high-priority case with no update today, or a frustrated customer.
   2. **Follow-up due today or overdue:** the cadence date has passed. Say which touch it is and which skill drafts it.
   3. **Status mismatch:** the status does not match who has the ball (e.g. Need More Information while we owe the next step, Solution Provided after the customer replied with an open issue). Refer to `/case-sp-triage` for the detailed move.
   4. **Closure candidates:** Solution Provided with the cadence exhausted, or Need More Information past 7 days with the 48-hour notice already sent.
   5. **Due soon:** the next contact falls within the next 2 business days.
   6. **On track:** no action today.
6. **Judgment pass.** Before writing, re-read the latest exchange of every case outside "On track" and ask whether the user would be surprised by the call. Don't guess: when the data is unclear or unreadable, move the case to a "Needs your review" line instead of forcing a bucket. Check once that nothing new arrived while you were reading: `SELECT Parent.CaseNumber, Incoming, MessageDate FROM EmailMessage WHERE ParentId IN (<Ids>) AND MessageDate > <time of the first pull>`.

7. **Call the existing skills** for the cases that need action, using the Skill tool. They are draft-only or read-only, so this is safe; never call anything that sends, posts or edits. Do this after the report is written, in order of urgency:
   - **First contact pending** (no outbound message yet, or the case is new): `case-first-contact <case>`.
   - **Follow-up due** on Need More Information: `case-reminder <case>`. On Working: `case-first-contact continue <case>`.
   - **Needs a handover:** `case-handover <case> <team>`, only when the user has named the target team. Never guess the team.
   - **Status mismatch or closure candidates:** call `case-sp-triage` once, passing only those case numbers, since it applies the detailed status rules.
   - **Case summary** (the user asks, or a handover is going out): `case-summary <case>`.
   - Call at most **3 case-specific skills** per run, most urgent first, unless the arguments say `all`. List the cases not covered so the user can ask for them. If the arguments say `report` or `only report`, skip this step.
   - Show each result under its case, under a heading with the skill name. If a skill is not installed or fails, say so in one line and move on. Some skills are slash-only and the model cannot call them (for example `case-first-contact` in the published version): in that case show the exact command for the user to type, such as `/case-first-contact <case>`. Never claim a draft was made when the call failed.

## Output

Start with one line of counts, e.g. "17 open cases: 2 act now, 3 follow-ups due, 1 status mismatch, 2 closure candidates, 9 on track." If the screen was read, add what it was ("Screen: list view 'sla', 8 cases") or why it was skipped.

Then one section per bucket, in the order above, skipping empty ones. For each case in the first four buckets:
- [CaseNumber](https://orgcs.lightning.force.com/lightning/r/Case/<Id>/view): Subject | Contact | Status / Sub_Status | SLA
- **Why:** one sentence with dates (e.g. "Last outbound Nov 6; 4 business days without reply; Need More Information cadence is 3").
- **Next:** the exact action and the skill to run (e.g. "Run /case-reminder 481546772").

"Due soon" and "On track" are one compact line per case. Close the report with an **Assumptions** line (business days, cadence source, SLA source, anything unreadable). No email bodies, no long paragraphs, no confidence scores. The drafts from step 7 come after the report, one block per skill.

In **Next:** lines, write the skill call the user would otherwise type (e.g. "Run /case-reminder 481546772") and say whether step 7 already ran it.
