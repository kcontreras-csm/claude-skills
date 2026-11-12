---
name: case-summary
description: Generate a "CASE AI SUMMARY GENERATED" handoff/summary for a Salesforce support case, given a case number (e.g. 476196596) or a case record Id (500...). Use when the user says "case summary", "summarize case", or "summary" along with a case number or Id (e.g. /case-summary 476196596).
---

Build a case summary in the template below for the case in the arguments. The argument can be a case number (digits, e.g. `476196596`) or a Case record Id (starts with `500`). Draft only; never write it to the case unless the user asks.

## Steps

1. Load `mcp__orgcs__soqlQuery` via ToolSearch (`select:mcp__orgcs__soqlQuery`) if needed.
2. Look up the case. Use `WHERE CaseNumber = '<arg>'` for a number, or `WHERE Id = '<arg>'` for a `500...` Id:
   `SELECT Id, CaseNumber, Subject, Status, Description, GHO__c, GHO_Type__c, Severity_Level__c, GEO__c, support_available_timezone__c, Customer_Temperature__c, Business_Issues__c, AX_Issue_Resolution_Summary__c, Account.Name, Account.Phone, Contact.Name, Owner.Name, CreatedDate FROM Case WHERE ... LIMIT 1`
3. Get the case history using the case Id:
   - `SELECT Subject, TextBody, Incoming, MessageDate, FromAddress FROM EmailMessage WHERE ParentId = '<Id>' ORDER BY MessageDate ASC LIMIT 30`
   - `SELECT CommentBody, CreatedBy.Name, CreatedDate, IsPublished FROM CaseComment WHERE ParentId = '<Id>' ORDER BY CreatedDate ASC LIMIT 30`
   - `SELECT Type, Body, CreatedBy.Name, CreatedDate FROM CaseFeed WHERE ParentId = '<Id>' ORDER BY CreatedDate ASC LIMIT 30`
   If the case has no emails, use the Description, feed posts, and `AX_Issue_Resolution_Summary__c` as the history, and say so.
   If email bodies are long, focus on the newest text in each and skip quoted reply chains and signatures.
4. Read the whole thread and fill the template. Base everything on the case data; don't invent facts.

## Field mapping

- GHO Type: `GHO_Type__c`. Leave blank if empty (and GHO__c is false).
- Severity: `Severity_Level__c`.
- From Geo: `GEO__c`. If empty, use the "Account Region" from the case-assignment feed post, else AMER (the case owner, Kevin, is in AMER), and flag it as assumed.
- To Geo (AMER/EMEA/APAC/JP/IST): leave blank unless the user says where it's going.
- Account Name: `Account.Name`.
- Account Phone: `Account.Phone`, formatted like `(512)-288-8511`. Use an extension only if one appears in the case.
- Hours: `support_available_timezone__c` (plus any availability hours the customer stated in emails).
- Customer Contact [Full Name]: `Contact.Name`.
- Executive Name [Internal]: leave blank unless an internal executive is named in the case.
- Problem Statement / Issue Details: 1-3 sentences on what the customer reported, with key dates, IDs and error messages.
- Customer Temperature / Pulse: `Customer_Temperature__c` if set; otherwise infer Positive / Neutral / Negative from the customer's tone and flag that it was inferred.
- Business Impact: `Business_Issues__c` if set; otherwise one sentence inferred from the issue.
- Current Actions: numbered list, in order, of what support and the customer did so far (include IDs such as Cron Trigger IDs, article links, internal requests).
- Next Steps: what the customer and support need to do next, including what to provide if the issue happens again. End with "The case is currently assigned to <Owner.Name>."
- Reference Documents: links to help articles, known issues, or docs shared in the case. Leave blank if none.

## Template

Reproduce exactly, including the header line and the `**bold**` labels in the single paragraph:

--CASE AI SUMMARY GENERATED--

GHO Type: <value>

Severity: <value>

From Geo: <value>

To Geo (AMER/EMEA/APAC/JP/IST): <value>

Account Name: <value>

Account Phone: <value>

Hours: <value>

Customer Contact [Full Name]: <value>

Executive Name [Internal]: <value>

**Problem Statement / Issue Details:** <text> **Customer Temperature / Pulse:** <value> **Business Impact:** <text> **Current Actions:** 1. <action> 2. <action> ... **Next Steps:** <text> The case is currently assigned to <Owner.Name>.

Reference Documents: <links>

## Output

Show the case number, subject, status, and owner in one line, then the summary in plain text (never in a code block or blockquote) so it can be copied straight into the case. After it, list briefly any fields you left blank or inferred (for example From Geo assumed AMER, temperature inferred).

Always copy the summary to the clipboard, from the `--CASE AI SUMMARY GENERATED--` line through `Reference Documents:` (not the header line or the notes), using `pbcopy` with a quoted heredoc (`pbcopy <<'EOF' ... EOF`) so special characters aren't expanded. Confirm with `pbpaste | head -3`, then tell the user it's on the clipboard.
