---
name: Case Closure
version: 1.0.0
description: Draft a short case-closure notice email for a Salesforce support case number when the customer hasn't replied to our earlier messages or reminders. Use when the user says "close the case", "closure email", "closing notice", or "no response close" along with a case number (e.g. /case-closure 477221865).
---

Draft a plain closure-notice email for the case number in the arguments. Draft only; never send, and never change the case status.

## Steps

1. Look up the case with `mcp__orgcs__soqlQuery` (load via ToolSearch `select:mcp__orgcs__soqlQuery` if needed):
   `SELECT Id, CaseNumber, Subject, Status, Description, Contact.Name, Contact.Email, Account.Name FROM Case WHERE CaseNumber = '<number>' LIMIT 1`
2. Get the latest emails and published comments using the case Id:
   - `SELECT Subject, TextBody, Incoming, MessageDate FROM EmailMessage WHERE ParentId = '<Id>' ORDER BY MessageDate DESC LIMIT 5`
   - `SELECT CommentBody, CreatedBy.Name, CreatedDate FROM CaseComment WHERE ParentId = '<Id>' AND IsPublished = true ORDER BY CreatedDate DESC LIMIT 5`
3. Find our outbound messages (`Incoming = false`): the original solution or questions and any reminders, with dates. Ignore customer emoji reactions ("reacted to your message"); they are not real replies. Check both incoming emails and customer case comments (comments starting with `<Created By: <contact name>>`) for anything newer than our last outbound message.
4. Pick the mode:
   - **Customer confirmed**: the customer's latest message says the issue is resolved or asks us to close the case (for example "it worked", "this case can be closed"). Draft the confirmation closure (see below). If the latest message still reports a problem, don't use this mode.
   - **No response**: the customer hasn't replied since our last outbound message. If no reminder has been sent yet, say so and suggest /case-reminder first, but still draft the closure notice if the user wants it.
   - **Customer replied with an open issue**: say so and ask the user how to proceed instead of drafting a closure notice.
5. Draft the email.

## Email format (confirmation closure)

- Subject and greeting: same as below.
- Thank them for confirming that <short description of what now works>, and say we're closing the case as they requested.
- Optionally add one short prevention tip from our earlier emails (for example a setting to avoid or a key article link).
- Say that if the issue comes back, they can open a new case through the Help portal and reference this case number, plus the one detail to include (for example a Cron Trigger ID or the exact error message). Don't invite a reply to this email.
- Thank them. Keep it to 2 or 3 short paragraphs.
- Same signature as below.

## Email format (no response)

- Subject: `RE: <case Subject> [ ref:... ]`. Reuse the exact `ref:` token from the previous outbound email subject.
- Greeting with the contact's first name.
- One sentence: we haven't heard back on case <number> since we sent <short recap> on <date> (and the reminder on <date>, if any).
- State that we are closing the case now. If we offered a solution, say we hope it helped.
- Avoid reopenings: never invite them to reply to reopen the case, and don't ask a question that would prompt a reply. Instead, say that if they need help in the future, they can open a new case through the Help portal and reference this case number. If useful, name the one detail to include in the new case (for example the exact error message).
- Thank them. Keep it to 3 or 4 short paragraphs, with no step-by-step recap.
- Sign as:

  Kevin Alexis Contreras
  Success Engineer | Salesforce

## Output

Show the case status, contact, and last outbound date in one line, then the draft in plain text (never in a code block or blockquote). Copy the draft body (without the Subject line) to the clipboard with pbcopy. Remind the user to update the case status to closed manually after sending.
