---
name: case-first-contact
disable-model-invocation: true
description: OrgCS case first-contact prep. Given a case number (e.g. 476196596) or a case record Id (500...), pulls the case (incl. Product & Topic and Product Feature), checks similar cases, known issues, Knowledge articles and the Lessons Learned log, decides scope / transfer / swarm-lead fast path, and drafts a short first-contact message (or a follow-up in "continue" mode). Use when the user says "first contact", "prep case", "new case", "first-contact prep", or "continue case" along with a case number (e.g. /case-first-contact 476196596, /case-first-contact continue 476196596).
---

Prep the case in the arguments for first contact. The argument can be a case number (digits), a Case record Id (starts with `500`), or a Lightning link (take the `500...` Id from the URL). If the arguments include "continue" / "follow up", run the **Continuing the Case** flow instead of first contact. Draft only: never send emails, post comments, or edit the case or the Lessons Learned log unless the user explicitly asks.

## Tools (Claude Code)

- Load via ToolSearch if needed: `select:mcp__orgcs__soqlQuery,mcp__orgcs__find,mcp__orgcs__getUserInfo` (OrgCS and H&T org, org id 00D30000000XsfGEAS).
  - SOQL → `mcp__orgcs__soqlQuery`. SOSL → `mcp__orgcs__find`.
  - Current user's own name and role (for the message signature) → `mcp__orgcs__getUserInfo`. Never use a name from a prior run or an example.
- If CSMOrgCSMCP / CloudSuccessCaseMCP tools are available in this session (check with ToolSearch) and expose case-search or knowledge-search functions, try those FIRST for steps 5-7. If unavailable, erroring, or returning zero rows, you MUST still run the SOQL/SOSL fallback searches. An MCP zero-row result never replaces the fallback.
- Internal Knowledge articles linked below (Guidebook, Gen Usage troubleshooting, How-To scope): read their content with SOQL on `Knowledge__kav` by Id (e.g. `SELECT Id, Title, UrlName, Summary FROM Knowledge__kav WHERE Id = 'ka0Hx000002yxC9IAI'`; use `getObjectSchema` on Knowledge__kav if you need the body field name). If unreadable, say so and continue with the general checklist below.
- Lessons Learned log (Slack canvas): there is no Slack tool here. Try `select:mcp__satellite__list_open_pages,mcp__satellite__read_page` and read the canvas if it's open in Satellite. Otherwise report "Lessons Learned log: not accessible" in Searches Run and suggest the user open it (https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5). Never claim it was checked when it wasn't.
- Writing a Lessons Learned entry: draft it in the output for the user to paste into the canvas; don't try to write to Slack.
- Run independent searches in parallel. Case link format: `https://orgcs.lightning.force.com/lightning/r/Case/<Id>/view`; article link: `https://orgcs.lightning.force.com/lightning/r/Knowledge__kav/<Id>/view`.

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

**If the topic is unclear or the match is doubtful:** do not label it a transfer candidate. Say the topic is unclear and run the normal flow with a holding draft.

## Steps

1. **Retrieve the case** — Query the Case record by case number from the OrgCS and H&T org: Subject, Description, Status, Priority, CaseRoutingTaxonomy__c (Product & Topic), CaseReportingTaxonomy__c (Product Feature), GEO__c, Product, AccountId, ContactId, Escalation fields, Origin, CreatedDate, IsEscalated. Also check for attachments (logs, screenshots) — note if any exist. If the case already has a CaseReportingTaxonomy__c (Product Feature) value, carry it into the summary as-is. If it's blank, suggest a Product Feature only from the CaseReportingTaxonomy__c records that are valid, selectable options for this case's Product & Topic (CaseRoutingTaxonomy__c) — query CaseReportingTaxonomy__c filtered to that Product & Topic and pick the best-matching one, never a free-text guess. Label it clearly as "Suggested" rather than a confirmed field value, and if no valid CaseReportingTaxonomy__c option matches, say "not specified" instead of inventing one.
2. **Summarize** — Write a 2-3 sentence plain-language summary of the issue: what's broken/asked, who's affected, since when.
3. **Escalation/SLA risk check** — Using IsEscalated, Priority, and case age (CreatedDate vs. now), flag if the case is trending toward an SLA breach or already escalated. Call this out clearly at the top of the summary so it doesn't get missed.
4. **Prior conversation check** — Pull recent CaseComment/EmailMessage entries on the case and summarize the last few exchanges, so the draft doesn't repeat what's already been said or miss something the customer already answered.
5. **Similar cases (MANDATORY — always run, in-scope or out-of-scope)** — Extract 3-5 keywords from the Subject/Description and run at least two searches: (a) SOSL/SOQL LIKE on Subject/Description keywords, and (b) same AccountId and/or same CaseRoutingTaxonomy__c / FunctionalArea__c / Product. Exclude the current case. List up to 5 with case number, subject, status, and link. Record the queries run and the number of results.
6. **Related known issues (MANDATORY — always run)** — Search for linked Bug__c records (including the case's own Bug__c field) and known-issue cases tied to the same product/functional area/keywords. Record the queries run and the number of results.
7. **Knowledge articles (MANDATORY — always run, in-scope or out-of-scope)** — Search Knowledge__kav using the case's subject/keywords AND the Product & Topic / product area (at least two searches). Note the best match if any. For out-of-scope cases this result is required for the redirect message. Record the queries run and the number of results.
8. **Lessons Learned check** — Open the [OrgCS Case Lessons Learned Log](https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5) and look for entries matching this case's product feature or issue type (see Key Principles for how to cite them and how to treat AI-generated entries).
9. **Scope check** — Compare the case's nature (break/fix? how-to? which cloud?) against the Scope Definition above. Decide IN SCOPE or OUT OF SCOPE and state why in one line. Steps 5-8 have already been completed by this point regardless of the result.
10. **If IN SCOPE — gather troubleshooting details** — Open [Internal: General troubleshooting for any Gen Usage Issue](https://orgcs.lightning.force.com/lightning/r/Knowledge__kav/ka0Hx000002yxC9IAI/view) and work through its general troubleshooting questions/checklist against this specific case (e.g. environment, when it started, steps to reproduce, affected user/profile, browser, recent changes, error messages — whatever that article calls out). Cross-check against step 4's prior conversation so you don't re-ask something already answered. Record which of those details are already known and which are still missing. Then pick ONLY the 1-3 most essential missing details (hard maximum of 3) that are truly needed to move forward. Drop anything nice-to-have, anything you can find yourself in the case/org data, and anything already asked or answered in the case. This step is skipped for OUT OF SCOPE cases.
11. **Draft first-contact message** — Using the case's Contact name, the current user's own name and role (the person running this skill — never a name from a prior run or example), and the summary:
    1. If IN SCOPE: draft a message introducing the current user by their own name and "Success Engineer", naming who they're addressing (the case contact), stating in ONE sentence that they're looking into the issue, and — pulling from step 10 — asking for the 1-3 essential missing details. Follow the Customer Message Rules below (short, no recap of what the customer already knows, no filler questions).
    2. If OUT OF SCOPE: draft a message introducing the current user by their own name, explaining this falls outside the team's scope, and sharing the best-matching knowledge article link from step 7 (or noting none was found) so the contact/team can be redirected properly.

Steps 1-11 are the first-contact pass and should stay focused purely on getting an accurate, SLA-safe first message out — do not fold in the follow-up work below while drafting first contact. Do NOT skip, shorten, or merge steps 5, 6, 7 and 8: run every one of them before drafting, and never write "None found" for them without having actually run the searches. Report them in the **Searches Run** section of the output.

## Swarm Lead Fast Path (run after step 4, before step 10)

*Based on a sample of swarm requests from Aug to Oct 2026 in <#C0BSLMLPWCW>, <#C028BLR714Y> and Rafael Castro's announcements. Treat it as a starting point and refine it with new cases.*

Some cases need no investigation: only the swarm lead (today mainly Rafael Castro) can do the backend action. For those, skip step 10's troubleshooting and use the fast path. Steps 5-8 still run.

**Request types usually handled only by the swarm lead (with sample case numbers):**

| Type | Sample cases | Details the swarm needs |
| --- | --- | --- |
| Password reset email | 474601346, 474657676, 474745038 | Username or email, Org ID |
| Passkey or MFA reset for a user | 474667513, 474710405 | Usernames, Org ID |
| Unfreeze user or contact (incl. sandbox) | 475241279, 474710741, 474688995 | Username, User ID, Org ID, sandbox or production |
| Welcome or activation email resend | 474612303, 474676966, 474747092 | Email address or User ID, Org ID |
| Remove .invalid suffix | 474624448, 474730973, 475012318 | Username, User ID, Org ID, sandbox or production, whether a password reset is wanted |
| MFA or PRMFA exception (remove, extend, disable) | 474999779, 475002427, 474640737, 474712622 | Org ID, sandbox or production |
| Disable Email Change Verification | 474643134, 474714640 | Org ID, authorized domains |
| Weekly export reset, unblock export job | 474630937, 475386883 | Org ID |
| Activity archival increase | 474647154, 475315611 | Org ID, requested days, Blacktab check that the requester is a System Administrator |
| Web-to-Lead limit increase | 474651883, 474877628 | Org ID, current and requested limit |
| Temporary API limit increase | 475882959 | Rafael's checklist below |
| Recycle Bin activation, Trialforce extension | 474729304, 474698786 | Org ID |
| Assign DPOC | 474640761 | Org ID, contact email |

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

## Confidence Gate: 90% Rule for Drafts (applies to EVERY draft, overrides steps 10-11)

Purpose: avoid asking the customer unrelated or unnecessary questions, and give the team time to check the specific problem with a human, without letting the case reach an SLA breach.

Before presenting any draft, rate your confidence (0-100%) INTERNALLY that you know the right next step for THIS case. Never show the confidence level, percentage or this rating in the response. Base it only on evidence you actually found (case data, prior conversation, matching Knowledge article, Lessons Learned entry, known issue, or a Swarm Lead Fast Path match). Do not count a hunch as evidence.

**If confidence is below 90% - send the General Holding Draft:**

* A short message (30-60 words) that says we are working on the case and will reach out soon, so the SLA is met.
* No questions, no troubleshooting steps, no guesses about the cause.
* Under the draft, add a short note to the user running the skill that this is a holding message and what a human should check before sending a detailed reply (e.g. no matching article, unclear request, conflicting information). Suggest next steps such as opening a swarm, or asking a teammate who handled a similar case. Do not mention the confidence level or percentage.

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

Produce a single reply structured as:

**Case Summary**

* Case: [Case Number] — [Subject] ([link])
* Contact: [Name] | Account: [Account] | Status: [Status] | Priority: [Priority]
* Product & Topic: [value from CaseRoutingTaxonomy__c, or "not specified"]
* Product Feature: [value from CaseReportingTaxonomy__c, or "Suggested: [a valid CaseReportingTaxonomy__c option for this case's Product & Topic]" if blank, or "not specified" if no valid option matches]
* Summary: [2-3 sentence plain-language summary]

**Similar Cases** (or "None found")

* [Case Number] — [Subject] ([Status]) [link]

**Related Known Issues** (or "None found")

* [description / link]

**Knowledge Articles** (or "None found")

* [Title] ([link])

**Searches Run** (required — proves steps 5-8 were executed)

* Similar cases: [query/queries used] → [N results]
* Known issues: [query/queries used] → [N results]
* Knowledge articles: [query/queries used] → [N results]
* Lessons Learned log: [checked / not accessible] → [N matches]
* Tools used: [MCP tool(s) and/or SOQL/SOSL]

**Related Lessons Learned** (or "None found")

* [Case Number] — [Entry title] ([log link]) | Worked by: [teammate] | Human / AI-generated — [why it matches]. Suggest approaching [teammate] for context.

**Scope Determination:** IN SCOPE / OUT OF SCOPE — [one-line reason]

**Transfer Candidate** (only if the Topic Check says so): [Product & Topic found] — [one-line reason]

**Transfer message, optional internal case comment** (only for transfer candidates):

> [Draft internal comment, under 80 words]

**Draft First-Contact Message** (50-100 words, max 3 questions)

> [Ready-to-send draft message]

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

## Key Principles

* Before drafting, check the [OrgCS Case Lessons Learned Log](https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5) for an entry matching this case's product feature or issue type — reuse its eligibility rule/resolution if relevant, and cite it as a prior lesson rather than re-deriving from scratch. Each entry carries a "Worked by" line (the teammate who owned that case): in the first-contact output, list every matching entry with its case number, link and worked-by name, and suggest the user approach that teammate for context (e.g. "Alejandra Mata worked a similar case, 474xxxxxx"). If the matching teammate is the current user, say so instead of suggesting a contact.
* After a case is resolved, if the user shares the correct resolution, root cause, or an eligibility rule that wasn't obvious upfront, add an entry to the [OrgCS Case Lessons Learned Log](https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5) (newest on top): case number, Worked by (the case Owner's full name, so others know whom to ask), product feature, scope call, the rule/root cause, and the correct resolution. Skip this when the case was routine and taught nothing new.
* Lessons Learned entries tagged as AI-generated (created from case data without human input, e.g. by the OrgCS Lessons Learned Backfill skill) are lower-confidence than human-written ones. Cite them as a "possible prior lesson" to verify, not as a confirmed rule. If both an AI entry and a human entry exist for the same case or issue type, use the human entry.
* When adding a Lessons Learned entry for a case that already has an AI-generated entry, replace the AI entry with the human one and remove the AI-generated tag, rather than creating a duplicate.
* Never invent case data — if a field is missing, say "not specified" rather than guessing.
* Always link every case and article referenced using its Salesforce URL.
* Keep every draft message SHORT, warm, and professional (see Customer Message Rules: 50-100 words, max 3 questions, no repeated questions, no filler) — no jargon the customer wouldn't understand.
* If the case number can't be found, say so plainly and stop rather than guessing at a similar case.
