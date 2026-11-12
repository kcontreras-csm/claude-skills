---
name: Case First Contact
version: 1.1.0
disable-model-invocation: true
description: OrgCS case first-contact prep. Given a case number (e.g. 476196596) or a case record Id (500...), pulls the case (incl. Product & Topic and Product Feature), checks similar cases, known issues, Knowledge articles and the Lessons Learned log, decides scope / transfer / swarm-lead fast path, and drafts a short first-contact message (or a follow-up in "continue" mode). Use when the user says "first contact", "prep case", "new case", "first-contact prep", or "continue case" along with a case number (e.g. /case-first-contact 476196596, /case-first-contact continue 476196596).
---

Credits: based on the "OrgCS Case First-Contact Prep" skill by Abraham Zaid Aguilar (Slack canvas). The Claude Code port, including the Tools section and the OrgCS MCP and Satellite adaptations, is by Kevin Alexis Contreras. Content synced with the canvas on 2026-11-12.

Prep the case in the arguments for first contact. The argument can be a case number (digits), a Case record Id (starts with `500`), or a Lightning link (take the `500...` Id from the URL). If the arguments include "continue" / "follow up", run the **Continuing the Case** flow instead of first contact. Draft only: never send emails, post comments, or edit the case or the Lessons Learned log unless the user explicitly asks.

## Tools (Claude Code)

- Load via ToolSearch if needed: `select:mcp__orgcs__soqlQuery,mcp__orgcs__find,mcp__orgcs__getUserInfo` (OrgCS and H&T org, org id 00D30000000XsfGEAS).
  - SOQL → `mcp__orgcs__soqlQuery`. SOSL → `mcp__orgcs__find`.
  - Current user's own name and role (for the message signature) → `mcp__orgcs__getUserInfo`. Never use a name from a prior run or an example.
- If CSMOrgCSMCP / CloudSuccessCaseMCP tools are available in this session (check with ToolSearch) and expose case-search or knowledge-search functions, try those FIRST for steps 5-7. If unavailable, erroring, or returning zero rows, you MUST still run the SOQL/SOSL fallback searches. An MCP zero-row result never replaces the fallback.
- Internal Knowledge articles linked below (Guidebook, Gen Usage troubleshooting, How-To scope): read their content with SOQL on `Knowledge__kav` by Id (e.g. `SELECT Id, Title, UrlName, Summary FROM Knowledge__kav WHERE Id = 'ka0Hx000002yxC9IAI'`; use `getObjectSchema` on Knowledge__kav if you need the body field name). If unreadable, say so and continue with the general checklist below.
- Lessons Learned log (Slack canvas): there is no Slack tool here. Try `select:mcp__satellite__list_open_pages,mcp__satellite__read_page` and read the canvas if it's open in Satellite. Otherwise report "Lessons Learned log: not accessible" in Searches Run and suggest the user open it (https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5). Never claim it was checked when it wasn't.
- Writing a Lessons Learned entry: draft it in the output for the user to paste into the canvas; don't try to write to Slack.
- Swarm channels (Swarm Channel Lookout, step 8): if Slack search tools are available in this session, use them. If not, try `select:mcp__satellite__list_open_pages,mcp__satellite__read_page` for an open Slack page; otherwise write "Swarm lookout: not run (no Slack access)" in Searches Run, give the user the exact queries and channels to search by hand, and never write "None found" for searches that were not run.
- Run independent searches in parallel. Case link format: `https://orgcs.lightning.force.com/lightning/r/Case/<Id>/view`; article link: `https://orgcs.lightning.force.com/lightning/r/Knowledge__kav/<Id>/view`.

## How It Works

**ALWAYS (standalone):** Uses Salesforce SOQL/SOSL against the OrgCS and H&T org (org id 00D30000000XsfGEAS) to pull the case, search similar cases, and search Knowledge__kav articles.

**CONNECTED (if available):** If CSMOrgCSMCP / CloudSuccessCaseMCP tools are enabled and expose case-search or knowledge-search functions, try those FIRST for steps 5-7 (similar cases, known issues, knowledge articles), since they're purpose-built for this. If an MCP tool is unavailable, errors out, or returns no results, you MUST fall back to the standalone SOQL/SOSL searches described in those steps. An MCP result of zero rows never replaces running the fallback search.

## Scope Definition (Success Engineer)

A case is IN SCOPE if it is either:

* A **break/fix** issue (something broken/erroring in the product), OR
* A **Sales Cloud or Service Cloud "How-To"** case, per [INTERNAL: What is a Sales and Service How-To case?](https://orgcs.lightning.force.com/lightning/r/Knowledge__kav/ka0Hx000000CjXdIAK/view) — i.e. a "how do I configure/use X" question in Sales or Service Cloud, OR
* An **org lifecycle status question** (org Expired/Hold/Locked/Deleted, revival or reactivation requests, data export from a locked org) per [Access to Organizations with 'Expired,' 'Hold' and 'Locked' Status](https://help.salesforce.com/s/articleView?id=000386225&type=1) — confirmed in scope per Jorge Andres Santos.

A case is OUT OF SCOPE if it's a how-to/config question for a cloud/product outside Sales/Service Cloud, or clearly not break/fix and not sales-related.

Reference articles to use every time:

* [Internal_Support_OrgCS Guidebook Case Handling](https://orgcs.lightning.force.com/lightning/r/Knowledge__kav/ka0Hx000002t1WzIAI/view) — general case navigation/handling guidance.
* [Internal: General troubleshooting for any Gen Usage Issue](https://orgcs.lightning.force.com/lightning/r/Knowledge__kav/ka0Hx000002yxC9IAI/view) — **the most important reference**. Always pull this article's troubleshooting questions/checklist and use it to shape the investigation angle in the draft message (what to ask/check first).

## Topic Check and Case Transfer (run right after step 1)

*Source: Luis Eduardo Naranjo, Oct 23, 2026. Update this list when the team's topics change.*

Topics our team works (Product & Topic from CaseRoutingTaxonomy__c):

* **Sales Gen Usage:** all objects, data exports, how-to.
* **Security and Activations.**
* **Dev Support:** optional. Treat it as workable, but mention in the note that the user can choose to take it or transfer it.

**If the case's Product & Topic is one of these:** continue the normal flow.

**If it is NOT one of these (or the topic is blank and the subject clearly points elsewhere):**

* Label the case **Transfer candidate** in the output, with the Product & Topic found and a one-line reason.
* Still run steps 5-8 and report the SLA risk from step 3. Do not skip them.
* Customer draft: a short holding message only (see Confidence Gate). No troubleshooting questions.
* Also draft a **Transfer message (optional internal case comment)** the user can paste as an internal CaseComment (not visible to the customer). Keep it under 80 words and include: case number, Product & Topic found, why it is outside our topics, what the customer needs (one line), key details already known (Org ID, user, error), and what has been checked so far. Do not invent a destination team; if the right queue is not clear, write "destination: to be confirmed".
* The user decides whether to transfer and whether to post the comment. Never post it yourself.

* **Handover template (suggest only):** when the case is a transfer candidate, or a transfer is confirmed (by the user or a swarm lead), tell the user to run the `/case-handover <case> <team>` skill to get the standard HANDOVER DETAILS template. Do not run it, copy its template, or call it from this skill. Remind the user to have ready: case number, Production or Sandbox, Org ID, affected cloud, target team and the reason for routing.

**If the topic is unclear or the match is doubtful:** do not label it a transfer candidate. Say the topic is unclear and run the normal flow with a holding draft.

## Steps

1. **Retrieve the case and confirm the Org ID.** Query the Case by case number from the OrgCS and H&T org. Pull Subject, Description, Status, Priority, Product & Topic (CaseRoutingTaxonomy__c), Product Feature (CaseReportingTaxonomy__c), GEO__c, Product, AccountId, ContactId, escalation fields, Origin, CreatedDate and IsEscalated. Also check for attachments (logs, screenshots) and note if any exist. ORG ID: retrieve the customer's Org ID from the case (or from the Account if the case does not have it; find the field that holds it at run time), show it at the top of the Case Summary, and ask the user to confirm it is the org we are working in (Production or Sandbox) before any troubleshooting or backend action. If it is missing or values conflict, say so and never guess. LANGUAGE: detect the customer's language from the Description and emails; write drafts in that language and show it in the summary. MOOD: read the tone of the customer's messages; if frustrated or aggressive, flag it at the top, open the draft by acknowledging the frustration and the wait, stay calm and professional, and never mirror the tone. PRIOR RESPONSES: check whether anyone has already replied to the customer (outbound emails or public comments). If so, label the run "Not a first contact (already responded)" and list who replied and when; we still introduce ourselves in the draft (see step 12). PRODUCT FEATURE: if CaseReportingTaxonomy__c has a value, carry it as-is. If blank, suggest only a valid CaseReportingTaxonomy__c option for this Product & Topic, labeled "Suggested"; if none matches, write "not specified".
2. **Summarize** — Write a 2-3 sentence plain-language summary: what is broken or asked, who is affected, since when, and the Org ID (Production or Sandbox). Add one short line with the detected customer language and mood (calm, concerned or frustrated). If the case has prior responses, add "Not a first contact" with who replied and when.
3. **Escalation/SLA risk check** — Using IsEscalated, Priority, and case age (CreatedDate vs. now), flag if the case is trending toward an SLA breach or already escalated. Call this out clearly at the top of the summary so it doesn't get missed.
4. **Prior conversation check** — Pull ALL available CaseComment, EmailMessage and Task-logged email entries on the case, not only recent ones. Summarize the exchanges in chronological order and note who sent each reply (the user or another engineer) and the date. Then show a **Questions Tracker** table with: question asked (by whom, date), status (Answered, Unanswered or Partially answered), and the customer's answer if any. Step 11 uses it to skip questions already asked or answered. This feeds the "Not a first contact" label and the choice of opening (intro or no intro). Read the customer's tone for the mood check. If any data cannot be read because of the security policy, say so instead of assuming there is none.
5. **Similar cases (MANDATORY — always run, in-scope or out-of-scope)** — Extract 3-5 keywords from the Subject/Description and run at least two searches: (a) SOSL/SOQL LIKE on the keywords, and (b) same AccountId and/or same CaseRoutingTaxonomy__c / FunctionalArea__c / Product. Exclude the current case. List up to 5, same-account cases first, then same-topic cases. For each give case number, subject, status, link and owner name, plus a one-line resolution if closed. Prefer closed, resolved cases from the last 12 months. Record the queries run and the number of results.
6. **Related known issues (MANDATORY — always run)** — Search for linked Bug__c records (including the case's own Bug__c field) and known-issue cases tied to the same product/functional area/keywords. Apply the GUS work item check below. For every GUS item found, show a **GUS Verification table** with columns: W-number, Type, Status, Team, Assignee, Verdict (confirmed known issue / investigation only / closed without code change / not verified, GUS not readable). Assignee is internal only. Customer drafts may say only, in plain words, that a known issue or investigation exists. Never include W-numbers, comments, resolution text or other internal content, and never promise a fix or ETA. Never state fix status from case comments alone. Record the queries run and the number of results.
7. **Knowledge articles (MANDATORY — always run, in-scope or out-of-scope)** — Search Knowledge__kav using the case's subject/keywords AND the Product & Topic / product area (at least two searches). List up to 5 matches with title, internal link and a one-line note on why each fits; mark the best match. The internal Knowledge__kav link is NOT visible to customers: label each link "Internal link (not customer-visible)" and tell the user to open the article manually and find its public URL (help.salesforce.com) before sharing it. In customer drafts never paste the internal link; use the placeholder [PUBLIC ARTICLE URL: add after checking] next to the article title in the body, and never invent the public URL. For out-of-scope cases the best match feeds the redirect message the same way. Record the queries run and the number of results.
8. **Swarm channel lookout (MANDATORY)** — search the three swarm channels for the case number, title and main points, and read the threads that match (see "Swarm Channel Lookout" below). Report it in Searches Run and in the Swarm Channel Findings section of the output.
9. **Lessons Learned check** — Open the [OrgCS Case Lessons Learned Log](https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5) and look for entries matching the case's product feature, issue type, request type and Org ID (Production or Sandbox). Rank human-written entries first, then AI-generated ones; for each give case number, link and "Worked by". Human entries: cite as a prior lesson and suggest approaching the Worked by teammate (if that is the current user, say so). AI-generated entries: do not use as a rule; show only as "You can check with [teammate]" (lower confidence). If a human entry exists for the same case or issue type, use it. If there is no match and the case is a new or non-routine topic, suggest adding a new entry after resolution. If the log cannot be opened, write "not accessible" and that no match was checked; never write "None found" for a log you could not read.
10. **Scope check** — Compare the case against the Scope Definition above. State IN SCOPE or OUT OF SCOPE with a one-line reason, and say whether it is a Swarm Lead Fast Path match. If the scope is doubtful, do not decide alone: tell the user to check with a swarm lead (Rafael Castro or Jorge Santos; backup channel <#C028BLR714Y>) before sending anything detailed, use the holding draft (T3) in the meantime so the SLA is met, and give the user the "Resumen para el swarm lead" block (see Key Principles).
11. **If IN SCOPE — gather troubleshooting details** — Open [Internal: General troubleshooting for any Gen Usage Issue](https://orgcs.lightning.force.com/lightning/r/Knowledge__kav/ka0Hx000002yxC9IAI/view) and work through its checklist against this case. Cross-check against the Questions Tracker from step 4 and the details the swarm asked for in step 8. Show a visible **Known / Missing** list in the output: Known = each detail already available with where it came from (case, email, attachment, org data); Missing = each detail still needed. From Missing, pick ONLY the 1-3 most essential details (hard maximum of 3). Drop anything nice-to-have, anything findable in the case or org data, and anything marked Asked or Answered in the tracker. Skipped for OUT OF SCOPE cases.
12. **Draft the message** — Pick the matching template from "Message Templates" below and fill only its slots. Apply the Tone and Empathy rules and the new closing format. Write in the customer's detected language. Use the case Contact's first name and the current user's own name and role (never a name from a prior run or example). Opening rule: on a true first contact, or when someone else sent the earlier reply, use the intro opening; if the current user sent the previous message, open with "Hi [client name]," and no intro. If the customer is frustrated or aggressive, open by acknowledging the frustration and the wait. Mention known issues or investigations only in plain words (no W-numbers or internal content). Use the [PUBLIC ARTICLE URL: add after checking] placeholder for articles. If scope is doubtful, use T3 and note that a swarm lead should check.
    1. IN SCOPE: introduce the user, address the contact, state in ONE sentence that you are looking into the issue, and ask for the 1-3 essential missing details. Follow the Customer Message Rules.
    2. OUT OF SCOPE: introduce the user, explain this falls outside the team's scope, and share the best-matching article (with the public URL placeholder) or note that none was found.

Steps 1-11 are the first-contact pass and should stay focused purely on getting an accurate, SLA-safe first message out — do not fold in the follow-up work below while drafting first contact. Do NOT skip, shorten, or merge steps 5, 6, 7 and 8: run every one of them before drafting, and never write "None found" for them without having actually run the searches. Report them in the **Searches Run** section of the output.

**GUS work item check (applies to step 6 and every draft):** For any GUS work item (W-xxxxx) referenced in the case, in its Bug__c field, or in a related case's comments, look up the ADM_Work__c record in the GUS org (00DT0000000DpvcMAC) BEFORE drafting. Check Type, Status, Team, Assignee, and comments. Never call it an open bug, known issue, or fix status based only on what a case comment says. If the item is closed (e.g. "Closed - Resolved Without Code Change") or is an Investigation, say so in Related Known Issues and do not promise a fix or ETA to the customer. If GUS cannot be read, say so and use the General Holding Draft.

## Swarm Channel Lookout (step 8, MANDATORY)

Channels to search: <#C028BLR714Y> (GU LATAM/AMER swarm), <#C027XT6D648> (Security and Activations swarm), <#C0BSLMLPWCW> (Sales CDMX swarm). Search all three, every case, in scope or not.

**What to search (at least one query per channel, run in parallel):**

1. The case number (exact).
2. The Subject, trimmed to its 3-5 most distinctive words (error text, feature name, object, action requested).
3. The main points: error message or code, Product Feature, the action requested (e.g. "unfreeze", "MFA", "export"), and the Org ID if present.
4. Window: last 90 days first. If 0 relevant results, widen to 180 days. Include bot messages (swarm requests are often posted by a workflow).

**What to do with hits:**

* Read the full thread of every relevant hit, not only the first message. The answer is usually in the replies.
* Classify each hit: **Same case** (already swarmed, so do not open a duplicate and say who is handling it), **Same issue, other customer** (reuse the resolution; never name the other customer in a customer draft), or **Same request type** (confirm the Swarm Lead Fast Path details).
* Extract: date, who replied, the resolution or next step, and what details the swarm asked for. If the swarm already asked for specific details, use them in the question filter.
* If the resolution depends on a GUS item, apply the GUS work item check before using it.
* Treat Slack answers as leads, not verified facts. They do not raise the draft to the detailed version unless confirmed in a Knowledge article, a GUS item or the case record (see Verified evidence only).
* Cite the thread link for each finding.

**If nothing is found:** write "None found" with the queries and channels searched, and the window used. Never write "None found" without running the searches.

**Routing hint:** if the case is a swarm-lead request type, tell the user which channel fits best (security and activation: <#C027XT6D648>; settings and limits: <#C0BSLMLPWCW>, backup <#C028BLR714Y>), and list the details to include.

**Additional swarm rules:** never name another customer in a customer draft. Show who replied in each thread as a person to ask internally. If a hit is "Same case" (already swarmed), say so at the top of the output so the user does not open a duplicate. Mark any swarm resolution "Lead, to verify" until confirmed in a Knowledge article, a GUS item or the case record. Keep all three channels and the 90/180 day window until further notice.

## Swarm Lead Fast Path (run after step 4, before step 10)

*Based on a sample of swarm requests from Aug to Oct 2026 in <#C0BSLMLPWCW>, <#C028BLR714Y> and Rafael Castro's announcements. Treat it as a starting point and refine it with new cases.*

Some cases need no investigation: only the swarm lead (today mainly Rafael Castro) can do the backend action. For those, skip step 10's troubleshooting and use the fast path. Steps 5-8 still run.

**Request types usually handled only by the swarm lead (with sample case numbers):**

|Type|Sample cases|Details the swarm needs|
|  ---  |  ---  |  ---  |
|Password reset email|474601346, 474657676, 474745038|Username or email, Org ID|
|Passkey or MFA reset for a user|474667513, 474710405|Usernames, Org ID|
|Unfreeze user or contact (incl. sandbox)|475241279, 474710741, 474688995|Username, User ID, Org ID, sandbox or production|
|Welcome or activation email resend|474612303, 474676966, 474747092|Email address or User ID, Org ID|
|Remove .invalid suffix|474624448, 474730973, 475012318|Username, User ID, Org ID, sandbox or production, whether a password reset is wanted|
|MFA or PRMFA exception (remove, extend, disable)|474999779, 475002427, 474640737, 474712622|Org ID, sandbox or production|
|Disable Email Change Verification|474643134, 474714640|Org ID, authorized domains|
|Weekly export reset, unblock export job|474630937, 475386883|Org ID|
|Activity archival increase|474647154, 475315611|Org ID, requested days, Blacktab check that the requester is a System Administrator|
|Web-to-Lead limit increase|474651883, 474877628|Org ID, current and requested limit|
|Temporary API limit increase|475882959|Rafael's checklist below|
|Recycle Bin activation, Trialforce extension|474729304, 474698786|Org ID|
|Assign DPOC|474640761|Org ID, contact email|


**Rafael's checklist for temporary API limit increases** (ask only what the case does not already answer): authorized System Administrator? Usage Based Entitlement soft limit known? Org ID; the emergency and its business impact; extra calls needed; how the limit was calculated; how long it is needed (about two weeks max); what will prevent it from recurring.

**Do NOT use the fast path when:**

* The case is a question, not an action (e.g. "how do I deactivate users").
* The requester is not verified as the account owner or a System Administrator (check before any reset, MFA or passkey change).
* The org is Expired, Hold or Locked (use the normal flow and the lifecycle article).
* It is real troubleshooting (stuck sandbox refresh, matching rules, permission set groups, export download errors, limits that cannot be raised). These swarms usually get many replies.
* The match is unclear. Run the normal flow.

**Routing:** password, passkey, MFA and other security or activation requests go to Rafael Castro, or urgent or critical MFA and S&A requests to <#C027XT6D648>. Other settings and limits go to Rafael's swarm channel <#C0BSLMLPWCW>, with <#C028BLR714Y> as backup when he or Jorge are out. Confirm current channels first, because swarm channels have been changing. MFA extension rules were also changing around Oct 20, so check current guidance before advising on MFA exceptions.

**When the case matches, output:**

* Label: **Handled via Swarm Lead** (type matched, plus the closest sample case).
* **Draft message to the customer** (Customer Message Rules still apply):
    * If required details are missing, ask only for them (max 3, no repeats of what the case already asked).
    * If nothing is missing, send only a short "we are working on it" message with no questions.
* **Note to the user running the skill:** ask them to reach the swarm lead or open a swarm, and list the details to include: case number, Org ID, username or User ID, sandbox or production, and the exact action requested. Add caveats, e.g. "MFA exceptions can be changed only once".
* Still show Case Summary, Searches Run and Scope Determination as usual.

## Fallback: Security-Policy Blocks on SOQL (intermittent)

Some users intermittently get this error on SOQL against Case, CaseComment or EmailMessage: "The operation you requested isn't allowed due to a security policy in your organization". First seen 2026-10-22 on case 476058138. The block is intermittent: the same query can pass for one case and fail for another, and filtering by `Id` or `CaseNumber` is not reliably safer. Do not stop and do not assume the data is empty. Follow this order:

1. **Try SOQL once.** If it works, continue with the normal steps.
2. **If SOQL is blocked, use SOSL.** Run `FIND {<case number>} IN ALL FIELDS RETURNING Case(Id, CaseNumber, Subject, Status, Priority, Description, CreatedDate, IsEscalated)`. In the 2026-10-22 test this returned the full case even while SOQL on the same case failed. Do not retry the blocked SOQL in a loop.
3. **Keep running the other searches.** Knowledge__kav, CaseReportingTaxonomy__c, CaseRoutingTaxonomy__c, Case_Relationship__c (filter by `Case__c` = the case Id) and similar-case SOSL searches were not blocked. Run steps 5-8 as usual.
4. **Read the emails through Task.** Query `SELECT Id, Subject, Status, CreatedDate, Description FROM Task WHERE WhatId = '<case Id>' ORDER BY CreatedDate DESC LIMIT 10`. Each logged email is a Task whose Subject starts with "Email:" and whose Description holds the recipient and the full body, including the customer's replies. In the 2026-10-23 test on case 476058138 this returned 4 emails while EmailMessage was blocked. Use it for the prior conversation check (step 4) and to apply the question filter. Not verified: whether every email is logged as a Task, so do not treat the list as complete.
5. **Read the case timeline through CaseFeed.** Query `SELECT Id, Type, Body, CreatedDate FROM CaseFeed WHERE ParentId = '<case Id>' ORDER BY CreatedDate DESC LIMIT 10`. The Body of comments and emails comes back empty, but it shows that comments (`CaseCommentPost`), emails (`EmailMessageEvent`) and status changes happened, and a `TextPost` may hold an automatic case summary with the contact name, account and resolution. CaseFeed cannot be searched with SOSL; use SOQL. Run no more than 2 queries per call, because a 5-query batch timed out.
6. **Comment text is the one gap.** CaseComment text could not be read by any route we tried: SOQL is blocked, CaseFeed has an empty Body, and FeedComment is admin-only ("directly querying FeedComment is only supported for admin users"). Ask the user to paste the internal and public comments if they matter for the reply.
7. **Report what could not be read.** In the output, add a **Data Not Readable** line listing what was blocked and not recovered (typically CaseComment text, Account, Product & Topic, Product Feature, Bug__c). Write "not readable (security policy)" for each. Never write "none" or "empty" for data you could not read. A SOSL search that returns 0 rows for comments or emails is not proof that none exist.
8. **Do not draft blind.** If the emails (step 4) were read, run the question filter on them and draft normally, subject to the Confidence Gate. If neither the emails nor the case history could be read, do not guess: ask the user to paste the latest comments or emails, or give the General Holding Draft marked "UNVERIFIED: check the case history before sending". Do not invent a contact name; use the contact from the Task (WhoId or recipient address), the CaseFeed summary, or a placeholder.
9. **Tell the user in one line** that SOQL was blocked, which workaround was used (SOSL, Task, CaseFeed), and that they can report it to their Salesforce admin if it keeps happening. Only the admin can change the policy. Known ticket for reference: #18630506.
10. **Record it in Searches Run:** the SOQL query that failed, the SOSL, Task and CaseFeed queries used, and the number of results of each.

## Customer Message Rules (apply to EVERY draft: first contact, follow-up, resolution, escalation)

Customers prefer short replies. Every draft must follow these rules:

**Never include in a customer draft:** internal work IDs (W-numbers), internal bug or investigation details, or details about other customers' cases. Describe the status in plain words instead (for example "we are still investigating").

* **Length:** 50-100 words, maximum 120 (excluding greeting and signature). Never longer unless the user explicitly asks for a detailed message.
* **Structure:** greeting, one sentence of context or status, the ask or next step, sign-off. No multi-paragraph messages.
* **Questions:** maximum 3, and only ones that are blocking. Present them as a short numbered list so the customer can answer quickly. If only one question is needed, ask one. Zero questions is fine if the next step is on us.
* **Question filter:** before including any question, check that it (a) is truly required to make progress, (b) is not answerable from the case, org data or attachments, and (c) was not already asked or answered anywhere in the case history. If it fails any check, drop it.
* **No repeats:** never re-ask something already asked, even if it is still unanswered. Instead, reference it in one line ("Still need the error screenshot") and do not restate the other questions.
* **No recap:** do not restate the whole issue back to the customer. At most one short clause, and none in follow-ups.
* **No filler:** no generic troubleshooting checklists, no "please provide any additional information", no "let me know if you have any questions" padding, no long apologies or intros. One line of warmth is enough.
* **Plain language:** no internal jargon, no internal article names or links unless it is the redirect/solution link the customer needs.
* **Follow-ups:** even shorter (30-70 words). Lead with the status or answer, then only the still-outstanding blocking item(s). If nothing is blocking, send an update or next step with no questions.
* **Self-check:** before presenting the draft, count words and questions; if over the limits, cut until within them. State the word count and question count in one line under the draft.

## Tone and Empathy (apply to EVERY draft)

The customer should feel heard and helped, not processed. Keep the length limits, but write like a kind, competent person.

* **Open by acknowledging their situation in one short phrase tied to the actual impact** (blocked login, data not showing, deadline, repeated attempts). Use the impact, not a generic apology. Example: "I understand how disruptive it is when your team can't log in."
* **Use their name and warm, simple words.** Prefer "I'm sorry for the trouble" over "We apologize for any inconvenience". Use at most one apology per message, and none if nothing went wrong on our side.
* **Show ownership:** say "I'm looking into this now" or "I'll take care of this", not "the team will review".
* **Make asks easy and respectful:** "To move this forward, could you share:" and add a short reason when the ask might seem odd ("this helps me confirm the right org").
* **Urgent or escalated cases:** name the urgency plainly ("I see this is affecting your whole team, so I'm treating it as a priority") without promising a time or outcome.
* **Frustrated customers (multiple contacts, repeated asks, long wait):** acknowledge the wait or repetition first, then give the next step.
* **Close with a human, specific line**, not filler: "I'll update you as soon as I have news."
* **Avoid:** blame ("as you should have"), jargon, stiff phrases ("kindly", "as per", "please be advised"), over-promising, exclamation marks beyond one, and emoji unless the customer used one.
* **Language:** reply in the language the customer wrote in. Keep product and feature names as written.
* **Tone self-check:** the draft has one acknowledgment, one ownership phrase, no blame, no promises of date or fix. If any is missing, fix it before presenting.

## Confidence Gate: 90% Rule for Drafts (applies to EVERY draft, overrides steps 10-11)

Purpose: avoid asking the customer unrelated or unnecessary questions, and give the team time to check the specific problem with a human, without letting the case reach an SLA breach.

Before presenting any draft, rate your confidence (0-100%) INTERNALLY that you know the right next step for THIS case. Never show the confidence level, percentage or this rating in the response. Base it only on evidence you actually found (case data, prior conversation, matching Knowledge article, Lessons Learned entry, known issue, or a Swarm Lead Fast Path match). Do not count a hunch as evidence.

**Verified evidence only:** claims taken only from case comments, emails or Slack (for example "this is a known bug" or "a fix is planned") do not count as evidence until confirmed in the system of record (GUS work item, Known Issue article or the case record itself). Unconfirmed claims must never be stated to the customer as fact.

**If confidence is below 90% - send the General Holding Draft:**

* A short message (30-60 words) that says we are working on the case and will reach out soon, so the SLA is met.
* No questions, no troubleshooting steps, no guesses about the cause.
* Under the draft, add a short note to the user running the skill that this is a holding message and what a human should check before sending a detailed reply (e.g. no matching article, unclear request, conflicting information). Suggest next steps such as opening a swarm, or asking a teammate who handled a similar case. Do not mention the confidence level or percentage.

## Message Templates (use these; fill slots, do not rewrite the structure)

Goal: drafts stay consistent run to run. Choose ONE template. Keep the fixed lines as written (only translate to the customer's language). Fill only the [slots]. Do not add paragraphs. The Customer Message Rules and Tone and Empathy rules still apply.

**MANDATORY OPENING AND CLOSING (every template, every draft, no exceptions):**

Opening, always these two lines first:

> Hi [client name], This is [engineer name], [position of the engineer, usually Success Engineer].

Closing, always these two lines last, in this order and nothing after them:

```
[Closing phrase],
[Engineer full name] | [Position]
```

* **Closing phrase:** "Best regards," by default. Use "Thank you for your patience," when the customer has waited, the case is delayed, or the customer is frustrated (T3 holding drafts, T6 and T7 follow-ups, SLA-risk cases). Translate the phrase to the customer's language.
* **Engineer line:** the current user's real full name, then " | ", then their position (usually Success Engineer), e.g. Abraham Zaid Aguilar | Success Engineer. Never use a name from a prior run or example. Keep the phrase and the name line separate.
* This closing REPLACES the old "Best regards, [Engineer name] Success Engineer" shown in templates T1-T8.
* [engineer name] in the opening is the current user. [client name] is the case Contact's first name.
* Follow-ups (T6-T7) open only with "Hi [client name]," when the current user sent the previous message. If another engineer sent it, include the intro. A case that already has a response is not a first contact, but we still present ourselves when needed.
* In the final self-check, confirm the draft starts with the right opening and ends with the phrase line and the name line.

* **Asking the customer for Login Access (when we need to see the org):** Do NOT write "Setup > Login Access Policies" and do not call it "admin access". Use this route: the customer clicks their profile > Settings > Grant Account Login Access (approximate name; exact menu names are not verified), selects "Salesforce Support", sets the number of days (usually 3) and saves. Do not invent exact menu names beyond this.

**T1. First contact, in scope, details needed (confidence at or above 90%)**

> Hi [client name], This is [engineer name], [position].
> 
> [Empathy line tied to their impact]. I'm looking into this now.
> 
> To move forward, could you share:
> 
> 1. [Essential detail 1]
> 2. [Essential detail 2, optional]
> 3. [Essential detail 3, optional]
> 
> [Specific closing line, e.g. "I'll update you as soon as I have an answer."]
> 
> Best regards, [Engineer full name] | [Position]

**T2. First contact, in scope, nothing missing (we act next)**

> Hi [client name], This is [engineer name], [position].
> 
> [Empathy line tied to their impact]. I'm working on [one-clause description of the request].
> 
> [Next step on our side, no date or outcome promised]. I'll update you as soon as I have news.
> 
> Best regards, [Engineer full name] | [Position]

**T3. General Holding Draft (confidence below 90%, 30-60 words, no questions)**

> Hi [client name], This is [engineer name], [position].
> 
> Thank you for contacting us. I've started working on your case, I'm reviewing the details carefully, and I will follow up with you soon.
> 
> Thank you for your patience, [Engineer full name] | [Position]

**T4. Out of scope, redirect (clear article match)**

> Hi [client name], This is [engineer name], [position].
> 
> Thank you for reaching out. [Empathy line]. This request falls outside what our team can assist with, and the best next step is in this article: [Article title] [PUBLIC ARTICLE URL: add after checking].
> 
> [Where to go if the article does not solve it, e.g. the right support path]. Thank you for your understanding.
> 
> Best regards, [Engineer full name] | [Position]

**T5. Swarm-lead request, details needed**

> Hi [client name], This is [engineer name], [position].
> 
> [Empathy line]. I'm working on [action requested]. To process it, I need:
> 
> 1. [Detail 1, e.g. Org ID]
> 2. [Detail 2, e.g. username]
> 
> Once I have them, I'll take it from there and update you.
> 
> Best regards, [Engineer full name] | [Position]

**T6. Follow-up, still waiting on the customer (30-70 words)**

> Hi [client name],
> 
> Just checking in on your case. To continue, I still need [outstanding item, one line]. As soon as I have it, I'll keep moving this forward.
> 
> Thank you for your patience, [Engineer full name] | [Position]

**T7. Follow-up, status update, no questions**

> Hi [client name],
> 
> A quick update: [plain-language status, no internal IDs]. [Next step on our side]. I'll write again as soon as there's news.
> 
> Thank you for your patience, [Engineer full name] | [Position]

**T8. Resolution / next steps**

> Hi [client name], This is [engineer name], [position].
> 
> [Empathy line]. [What was done or the answer, in one or two sentences]. [Short steps if the customer must act, max 3].
> 
> Could you please try it and let me know if everything is working as expected? Once you confirm, I'll be glad to proceed with closing the case. If you still need anything, I'm happy to keep helping.
> 
> Best regards, [Engineer full name] | [Position]

**Template rules**

**Closing-the-case rule (applies to T8 and any resolution, final or wrap-up message):**

* Never announce that we are closing the ticket (no "we are closing this case", "I will close this case", "the case will be closed", or any auto-close deadline). ONE EXCEPTION (manager rule, Oct 26): on a Need More Info case with 7 days without a customer reply, the follow-up may tell the customer the case will close in 48 hours if there is no response. Use this only in that situation.
* Always kindly ask the customer to confirm the solution worked and that they agree to close the case, e.g. "Could you please confirm that everything is working as expected, so I can proceed with closing the case?"
* Closing happens only after the customer's confirmation. If they have not replied, a follow-up (T6) gently repeats the request for confirmation, without pressure or deadlines.
* Keep the tone warm and open: leave the door open for more help, and do not sound like we are in a hurry to close.
* Self-check: a resolution draft contains one explicit, polite request for confirmation and no statement that the case is being closed.

* Always state under the draft: template used (T1-T8), word count, question count.
* Every template ends with the two-line closing (phrase, then name | position); "Thank you for your patience," is the default in T3, T6 and T7 and when the customer is frustrated or the case is delayed. Opening rule: T1-T5 and T8 include the intro; T6-T7 omit it only if the current user sent the previous message, otherwise add the intro line.
* If two templates could fit, prefer the more cautious one (T3 over T1).
* Never invent slot values. If a slot has no data, switch templates instead of filling it with a guess.
* If the user asks for changes, edit slots or tone only, and keep the template structure so the draft does not drift.

**If confidence is 90% or above - send a more detailed draft:**

* Give the specific next step, answer or request, still within the Customer Message Rules (50-100 words, max 3 questions, question filter applies) and within the team's current restrictions (scope, Swarm Lead Fast Path rules, no sharing internal-only articles or links).
* Do not show the confidence level or percentage under the draft. The Searches Run section already shows the evidence.

**Rules:**

* When in doubt, choose the General Holding Draft. A holding message is better than an unrelated question.
* Never promise a fix, a date, or an outcome in the holding draft. Say only that we are working on it and will follow up.
* Never use the holding draft to hide a missing scope decision: still report the Scope Determination and the SLA risk from step 3. If the case is close to an SLA breach or escalated, say so at the top of the output.
* If the case is OUT OF SCOPE and the redirect article is a clear match, the redirect message can be sent at 90% or above; otherwise use the holding draft.
* For follow-up runs, the same gate applies. A holding follow-up is 30-50 words.
* The confidence rating is internal only. Never add a Confidence line, percentage or score to the Output.

## Output

Produce a single reply in this order (omit sections that do not apply):

**Top flags:** SLA / Escalation Risk, "Same case already swarmed", frustrated-customer flag, follow-up cadence alert (if any).

**Case Summary**

* Case: [Case Number] — [Subject] ([link])
* Org ID: [value] (Production/Sandbox) — please confirm this is the org we are working in
* Contact: [Name] | Account: [Account] | Status: [Status] | Priority: [Priority] | Support tier
* Language: [detected] | Mood: [calm/concerned/frustrated] | Prior responses: [Not a first contact — who and when, or none]
* Product & Topic: [value or "not specified"]
* Product Feature: [value, or "Suggested: ...", or "not specified"]
* Summary: [2-3 sentences]

**Questions Tracker** (table: question, asked by/date, status, answer)

**Known / Missing** (in-scope cases)

**Similar Cases** (up to 5, same-account first; or "None found")

**Related Known Issues** with the GUS Verification table (W-number, Type, Status, Team, Assignee, Verdict)

**Knowledge Articles** (internal links labeled "Internal link (not customer-visible)"; user must look up the public URL)

**Swarm Channel Findings** (or "None found" with queries and window; mark resolutions "Lead, to verify")

**Related Lessons Learned** (human first; AI entries as "You can check with [teammate]")

**Searches Run** (required): similar cases, known issues, knowledge articles, swarm channels (queries, window, results per channel), Lessons Learned log, tools used.

**Data Not Readable** (only if the security-policy fallback was used)

**Scope Determination:** IN SCOPE / OUT OF SCOPE — [reason]; if doubtful, check with a swarm lead.

**Transfer Candidate** and **Transfer message** (only for transfer candidates, under 80 words, internal comment)

**Resumen para el swarm lead** (when the answer rests only on previous cases, articles or general knowledge, when scope is doubtful, or when the case is non-routine)

**Draft First-Contact Message** (template T1-T5, 50-100 words, max 3 questions)

> [Ready-to-send draft built from the chosen template]

Template: [T#] | Words: [N] | Questions: [N]

## Continuing the Case (after first contact)

Once the first-contact message above has actually been sent and the customer has responded (or time has passed), the same skill can be re-invoked in "continue" mode to keep working the case efficiently:

1. **Re-pull the case** — refresh Status, Priority, IsEscalated, and any new CaseComment/EmailMessage entries since first contact.
2. **Reconcile missing details** — compare the customer's reply against the missing-details list from step 10 of first contact. Mark each item resolved, partially answered, or still outstanding.
3. **Re-run escalation/SLA check** — recheck age and escalation status; flag if it has moved closer to breach since first contact.
4. **Update similar cases / known issues / KB articles / Lessons Learned** — re-run the mandatory searches from steps 5-8 (same rules, including the Searches Run section) in case new similar cases, a fix, or a new lesson has appeared since first contact.
5. **Draft next message** — either:
    1. a short follow-up (30-70 words, per the Customer Message Rules) that references at most the 1-2 still-outstanding blocking items in one line each (never re-ask resolved items and never restate questions already asked; if nothing is blocking, give a status update with no questions), or
    2. if all details are in and a known issue/article/fix applies, a draft resolution or next-steps message, or
    3. if still stuck and out of runway, a draft escalation note summarizing the case, what's been tried, and why it needs escalation.

Use the same Output structure above for a continuation run, but title the message block "Draft Follow-Up Message" (or "Draft Escalation Message") instead of "Draft First-Contact Message", and add a **Still Missing** list showing what's still outstanding.

**Additions for continuation runs:**

* **Check case activity first:** before suggesting any draft, check all activity since the last message (new comments, emails, status or owner changes, escalation and SLA changes, swarm or SI notes). Show an "Activity since last contact" line with dates. If someone else already replied or acted, say so and adapt the draft.
* **Follow-up cadence (manager rule, Oct 26):** Working cases: update every 2 days (progress or documented blocker). Need More Info: follow up with the customer every 3 days; at 7 days with no reply, the draft tells the customer the case will close in 48 hours if there is no response (this is an explicit exception to the closing-the-case rule). Open Structured Investigation: case note every 3-4 days confirming the SI is active and the customer is informed. High priority: daily follow-up, no exceptions. More than 3 days without activity is an alert: flag it at the top. If stuck, suggest contacting the tech leads <@U07HLE875NV> or <@U07H15JQNB0>.
* Re-confirm the Org ID; update the Questions Tracker (Answered/Partial/Unanswered) and use it for the Still Missing list; refresh Known / Missing; re-read the customer's mood; keep the customer's language; apply the intro rule and the new closing; keep the customer-facing rules (no W-numbers, no GUS content, public URL placeholder).
* **Lessons Learned reminder:** when the customer confirms the fix, remind the user to add a human-written Lessons Learned entry (replacing any AI entry for the case). Skip for routine cases that taught nothing new.

## Key Principles

* Before drafting, check the [OrgCS Case Lessons Learned Log](https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5) for an entry matching this case's product feature or issue type — reuse its eligibility rule/resolution if relevant, and cite it as a prior lesson rather than re-deriving from scratch. Each entry carries a "Worked by" line (the teammate who owned that case): in the first-contact output, list every matching entry with its case number, link and worked-by name, and suggest the user approach that teammate for context (e.g. "Alejandra Mata worked a similar case, 474xxxxxx"). If the matching teammate is the current user, say so instead of suggesting a contact.
* After a case is resolved, if the user shares the correct resolution, root cause, or an eligibility rule that wasn't obvious upfront, add an entry to the [OrgCS Case Lessons Learned Log](https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5) (newest on top): case number, Worked by (the case Owner's full name, so others know whom to ask), product feature, scope call, the rule/root cause, and the correct resolution. Skip this when the case was routine and taught nothing new.

* Lessons Learned entries tagged as AI-generated (created from case data without human input, e.g. by the OrgCS Lessons Learned Backfill skill) are lower-confidence than human-written ones. Cite them as a "possible prior lesson" to verify, not as a confirmed rule. If both an AI entry and a human entry exist for the same case or issue type, use the human entry.
* When adding a Lessons Learned entry for a case that already has an AI-generated entry, replace the AI entry with the human one and remove the AI-generated tag, rather than creating a duplicate.

* Never invent case data — if a field is missing, say "not specified" rather than guessing.
* Always link every case and article referenced using its Salesforce URL.
* Keep every draft message SHORT, warm, and professional (see Customer Message Rules: 50-100 words, max 3 questions, no repeated questions, no filler) — no jargon the customer wouldn't understand.
* If the case number can't be found, say so plainly and stop rather than guessing at a similar case.

* **Always confirm with swarm leads when unsure.** Working or closing a case out of scope, or giving wrong information, causes real harm. Whenever the answer rests only on previous cases, articles or general knowledge (not on a Knowledge article, a GUS item or the case record), do not present it as certain; say so and suggest checking with a swarm lead (Rafael Castro or Jorge Santos).
* **Resumen para el swarm lead (always in Spanish, informal but respectful).** Add this paste-ready block in those cases, and also when the case is non-routine. Maximum 80 words, in this order: (1) case number, Product & Topic, Org ID (Prod/Sandbox); (2) what the customer needs, one line; (3) what I found (similar cases, articles, swarm threads) and how sure I am; (4) the specific question for the lead (scope, next step or resolution). Start with the placeholder "Hola [swarm lead]," — the engineer picks whom to send it to. Never post it for the user.
* Always confirm the Org ID. Write to the customer in their language. Never put internal links, W-numbers or GUS content in customer drafts. Follow the manager's follow-up cadence.