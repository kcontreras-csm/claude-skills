---
name: case-handover
description: Fill in the standard "HANDOVER DETAILS" template for a Salesforce support case being routed to another team, given a case number (e.g. 476196596) or a case record Id (500...) and the target team. Environment (Production/Sandbox) and Org ID or MID are optional and taken from the case when possible. Use when the user says "handover", "hand over", "route to", or "transfer to <team>" along with a case number (e.g. /case-handover 476196596 Marketing Cloud team).
---

Build a handover note in the template below for the case in the arguments. The argument can be a case number (digits, e.g. `476196596`), a Case record Id (starts with `500`), or a Lightning link (take the `500...` Id from the URL). The arguments may also include the target team, routing reason, and optional details such as the environment or Org ID/MID. Draft only; never write it to the case unless the user asks.

## Steps

1. Load `mcp__orgcs__soqlQuery` via ToolSearch (`select:mcp__orgcs__soqlQuery`) if needed.
2. Look up the case. Use `WHERE CaseNumber = '<arg>'` for a number, or `WHERE Id = '<arg>'` for a `500...` Id:
   `SELECT Id, CaseNumber, Subject, Status, Description, Environment__c, Instance_Type__c, Case_Origin_OrgID__c, CustomerOrgId__c, MID__c, Cloud__c, Sub_Cloud__c, cssf_Product_Topic_Name__c, AX_Issue_Resolution_Summary__c, Account.Name, Contact.Name, Owner.Name, CreatedDate FROM Case WHERE ... LIMIT 1`
   If several cases match, list up to 5 (number, subject, account) and ask which one.
3. Get the case history using the case Id:
   - `SELECT Subject, TextBody, Incoming, MessageDate, FromAddress FROM EmailMessage WHERE ParentId = '<Id>' ORDER BY MessageDate ASC LIMIT 30`
   - `SELECT CommentBody, CreatedBy.Name, CreatedDate, IsPublished FROM CaseComment WHERE ParentId = '<Id>' ORDER BY CreatedDate ASC LIMIT 30`
   - `SELECT Type, Body, CreatedBy.Name, CreatedDate FROM CaseFeed WHERE ParentId = '<Id>' ORDER BY CreatedDate ASC LIMIT 30`
   If the case has no emails, use the Description, feed posts, and `AX_Issue_Resolution_Summary__c` as the history, and say so.
   If email bodies are long, focus on the newest text in each and skip quoted reply chains and signatures.
   If Salesforce isn't available or returns nothing, use only the user's input and say so.
4. Check the required inputs: cloud/product, target team, and routing reason. Take them from the arguments first, then the case (see Field mapping). If any are still missing, ask for all of them in ONE message, then continue. Don't ask for anything else. The environment and Org ID/MID are optional: never ask for them; take them from the case, and leave them unknown if the case doesn't have them.
5. Read the whole thread and fill the template. Base everything on the user's input and the case data; don't invent facts.

## Field mapping

- Issue: 2-4 sentences written for a team with no context. Use the Description and the latest customer messages. Include the cloud/product (`Cloud__c`, `Sub_Cloud__c`, `cssf_Product_Topic_Name__c`), key error messages, IDs, and dates.
- Troubleshooting Performed: in order, the steps support and the customer took, from emails, comments, feed posts, `AX_Issue_Resolution_Summary__c`, and the user's input. Include IDs, article links, and internal requests. If there are none, use "No troubleshooting steps have been done."
- Steps to Reproduce: steps found in the case or given by the user. Otherwise use "Pending - please replicate".
- Reason for handover: "Need help from the <Target Team> to assist the customer". The target team comes from the user, written as they gave it (for example "Need help from the Marketing Cloud team to assist the customer").
- Why is this case being routed?: "The issue is related to <reason>", where `<reason>` is the routing reason or product area.
- a) Sandbox or Production: the user's value, else `Environment__c` or `Instance_Type__c`, else a clear statement in the Description, emails, or comments (for example the customer says it happens in their sandbox, or the Org ID/URL shown is a sandbox).
- b) Org ID or MID: the user's value, else `Case_Origin_OrgID__c`, `CustomerOrgId__c`, or `MID__c`, else an Org ID (15 or 18 characters starting with `00D`) or MID written in the Description, emails, or comments. If several different IDs appear, list them and flag it in the notes.
- c) Affected Users: users named by the user or in the case (name, username, or email). Default to `Contact.Name` only if the case shows the contact is the affected user.
- d) New implementation: Yes/No only if the user or the case states it clearly.
- e) When did this issue occur: the date the customer says the issue started. Don't use `CreatedDate` unless the customer says the issue started that day.
- f) Working previously: Yes/No only if the user or the case states it clearly.
- g) Customer changes: Yes/No only if the user or the case states it clearly.
- Any item not found in the input or the case stays "[Unknown - please confirm with customer]". Never guess or infer.
- If the user's value differs from the case record, use the user's value and flag it in the notes.

## Template

This is the original template, with its `xxxxxxx` placeholders written as named `<...>` placeholders. Reproduce it exactly: same header, same dashed line (37 dashes), same labels and question wording, same punctuation, same blank lines, and the same order. Replace only the placeholder parts:
- "Description of the issue to the next team" → the Issue text.
- "No troubleshooting steps have been done." → the troubleshooting steps, or keep it as is if there are none.
- "Pending - please replicate" → the repro steps, or keep it as is if there are none.
- Each `<...>` placeholder → the value (`<Target Team>`, `<reason>`, `<value>`).
- "Yes/No" → "Yes" or "No".
- Any placeholder with no known value → "[Unknown - please confirm with customer]".

Don't add, rename, reorder, or remove any line or label. Keep each answer on the same line as its label, and keep answers short: a plain value, with no extra notes in parentheses. Put explanations in the notes after the template, not inside it.

HANDOVER DETAILS
-------------------------------------
Issue: Description of the issue to the next team
Troubleshooting Performed: No troubleshooting steps have been done.
Steps to Reproduce: Pending - please replicate
Reason for handover: Need help from the <Target Team> to assist the customer
Why is this case being routed? The issue is related to <reason>

Additional Information:
a) Does this issue occur in Sandbox or Production? <value>
b) What is the affected Org ID or MID? <value>
c) Who are the affected Users to troubleshoot/investigate? <value>
d) Is this a new implementation? Yes/No
e) When did this issue occur? <value>
f) Was this working previously? Yes/No
g) Did the customer make any changes at their end? Yes/No

## Output

Show the case number, subject, status, and owner in one line, then the handover in plain text (never in a code block or blockquote) so it can be copied straight into the case. After it, list briefly the sources used (user input, case record, number of emails/comments/feed posts read), the fields left unknown, and any place where the user's input differed from the case.

Always copy the handover to the clipboard, from the `HANDOVER DETAILS` line through item g) (not the header line or the notes), using `pbcopy` with a quoted heredoc (`pbcopy <<'EOF' ... EOF`) so special characters aren't expanded. Confirm with `pbpaste | head -3`, then tell the user it's on the clipboard.
