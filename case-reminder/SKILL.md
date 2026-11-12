---
name: case-reminder
description: Draft a short customer reminder email for a Salesforce support case number, asking for an update on the last thing we sent or asked. Use when the user says "reminder", "follow up", or "nudge" along with a case number (e.g. /case-reminder 476168508).
---

Draft a plain reminder email for the case number in the arguments. Draft only; never send.

## Steps

1. Look up the case with `mcp__orgcs__soqlQuery` (load via ToolSearch `select:mcp__orgcs__soqlQuery` if needed):
   `SELECT Id, CaseNumber, Subject, Status, Description, Contact.Name, Contact.Email, Account.Name FROM Case WHERE CaseNumber = '<number>' LIMIT 1`
2. Get the latest emails and published comments using the case Id:
   - `SELECT Subject, TextBody, Incoming, MessageDate FROM EmailMessage WHERE ParentId = '<Id>' ORDER BY MessageDate DESC LIMIT 5`
   - `SELECT CommentBody, CreatedBy.Name, CreatedDate FROM CaseComment WHERE ParentId = '<Id>' AND IsPublished = true ORDER BY CreatedDate DESC LIMIT 5`
3. Find our last outbound message (`Incoming = false`) and what it asked or did. Ignore customer emoji reactions ("reacted to your message"); they are not real replies. If the customer replied substantively, say so and ask the user how to proceed instead of drafting a reminder.
4. Draft the email.

## Email format

- Subject: `RE: <case Subject> [ ref:... ]`. Reuse the exact `ref:` token from the previous outbound email subject.
- Greeting with the contact's first name.
- One line saying this is a reminder about the case (include the case number) and one short sentence recapping what we sent or did, with the date.
- Ask whether it worked or what the status is. If our last email asked a question (for example which objects they export), repeat that question briefly. Include any key link from our earlier email.
- If it's still failing, ask for the exact error message.
- Closing nudge: if no reply, we plan to close the case in a few days, and they can reply to reopen it. Mention to the user that this line is optional.
- Keep it short, with no step-by-step recap unless the user asks.
- Sign as:

  Kevin Alexis Contreras
  Success Engineer | Salesforce

## Output

Show the case status, contact, and last outbound date in one line, then the draft in plain text (never in a code block or blockquote). Flag any assumptions (for example unanswered questions from the original description).
