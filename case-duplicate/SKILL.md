---
name: Case Duplicate
version: 1.0.0
description: Handle a duplicate Salesforce support case following the MX Sales Cloud team process. Given the duplicate case number and the original case number, checks both cases in OrgCS, drafts the internal post for the duplicate case (requesting non-visible / remove VSSP and close as duplicate), and drafts the Slack message asking Juan, Mau or Jorge to remove the View in Self Service Portal flag. Use when the user says "duplicate", "duplicado", "caso duplicado", "close as duplicate", "non-VSSP" or "mark invisible" along with case numbers (e.g. /case-duplicate 481564430 481564429).
---

Process for duplicate cases, defined by Rafael Castro in #mx-sales-cloud-equipazo (Nov 12, 2026). Keep only the case we are actively working. The other one gets an internal post, has VSSP removed by a manager, and is closed as Duplicate.

Drafts only. Never post to the case, change its status or visibility, or send the Slack message without the user's explicit OK.

## Inputs

- First number: the **duplicate** case (the one to close).
- Second number: the **original** case (the one we keep working).
- If only one number is given, look for the other one (step 2). If you still can't tell which case is the duplicate, ask the user.

## Steps

1. Look up both cases with `mcp__orgcs__soqlQuery` (load via ToolSearch `select:mcp__orgcs__soqlQuery` if needed):
   `SELECT Id, CaseNumber, Subject, Status, CreatedDate, Owner.Name, Contact.Name, Account.Name, Description FROM Case WHERE CaseNumber IN ('<dup>', '<orig>')`
2. If only one number was given, look for likely duplicates with the same Account and a similar Subject that were created within a few days:
   `SELECT Id, CaseNumber, Subject, Status, CreatedDate, Owner.Name FROM Case WHERE AccountId = '<AccountId>' AND Id != '<Id>' AND CreatedDate = LAST_N_DAYS:14 ORDER BY CreatedDate DESC LIMIT 10`
3. Sanity check. Confirm that both cases have the same account (or contact) and describe the same issue. Keep the case that has our work on it: owned by the user, with outbound emails or comments, or in Working status. If the check suggests the user swapped the numbers, or the cases aren't really duplicates, say so and stop before drafting.
4. Get the user's manager:
   `SELECT Manager.Name FROM User WHERE Email = 'kcontreras@salesforce.com' LIMIT 1`
   (Currently Mauricio Escalona.) The swarm lead is Rafael Castro unless the user names someone else.
5. Draft the two pieces below.

## Internal post (on the duplicate case)

[#INTERNAL] Case is duplicate from original case: <original case number> hence requesting this case to be marked as non-visible and closing it as duplicate
CC: <Manager name> <Swarm lead name>

## Slack message (to #mx-sales-cloud-equipazo, or a DM to Juan Ponce / Mauricio Escalona / Jorge Andres Santos)

Short and in Spanish. Name the **duplicate** case number(s) to mark as non-visible, not the original. Juan asked for exactly that. Example:

Hola Mau / Juan, ¿me ayudan a quitar el VSSP (marcar como non-visible) del caso <duplicate> por favor? Es duplicado del <original>, que es el que estoy trabajando. Ya dejé el post interno. ¡Gracias!

If the user asks to send it, load `mcp__plugin_slack_slack__slack_send_message` and send only after they confirm. #mx-sales-cloud-equipazo is C0BD5B8T65N.

## Output

1. One line per case: number, status, owner, account, created date, and which one is the duplicate.
2. The internal post in plain text (never in a code block or blockquote). Copy it to the clipboard with pbcopy.
3. The Slack message in plain text.
4. Remaining manual steps: paste the internal post on the duplicate case, wait for the manager to remove VSSP, then close the duplicate case with status/reason Duplicate.
