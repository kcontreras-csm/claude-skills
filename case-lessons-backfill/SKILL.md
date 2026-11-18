---
name: Case Lessons Learned Backfill
version: 1.0.0
description: Draft AI-tagged entries for the OrgCS Case Lessons Learned Log for closed cases where nobody wrote one, from case data only (emails, comments, feed posts and the swarm threads). Use when the user says "lessons learned backfill", "backfill the log", "llenar lessons learned" or "entradas pendientes del log". Draft only: it shows the proposed entries for the user to review and paste into the log; it never writes to Slack.
---

Credits: based on the "OrgCS Lessons Learned Backfill (AI-Drafted Entries)" skill by Abraham Zaid Aguilar (Slackbot). Companion to the case-first-contact skill, which reads these entries.

Backfill the OrgCS Case Lessons Learned Log for closed cases where nobody wrote an entry. Entries are drafted from case data only and are always tagged as AI-generated. Draft only: show the entries for review and never write to the log or to Slack.

## Tools (Claude Code)

- Load via ToolSearch `select:mcp__orgcs__soqlQuery,mcp__orgcs__find,mcp__orgcs__getUserInfo` (OrgCS and H&T org, org id 00D30000000XsfGEAS).
- Log: [OrgCS Case Lessons Learned Log](https://salesforce.enterprise.slack.com/docs/T01G0063H29/F0C50BQA3M5). Read it with a Slack tool if the session has one, or with `select:mcp__satellite__list_open_pages,mcp__satellite__read_page` if it is open in Satellite. If it cannot be read, say "Lessons Learned log: not accessible" and ask the user to paste the 2-3 newest entries and the entries for the cases in scope. Never claim the log was checked when it was not.
- Swarm channels: if Slack search tools are available, use them. If not, write "Swarm lookout: not run (no Slack access)", give the user the exact queries and channels to search by hand, and never write "no swarm thread found" for searches that were not run.
- If SOQL is blocked by the security policy, use the fallback in the case-first-contact skill (SOSL for the case, `Task` records for emails, `CaseFeed` for the timeline) and mark unreadable data as "not readable (security policy)".

## Inputs

- **Manager:** by default the current user's manager, from `getUserInfo`. Only resolve by name if the user names a different manager. Several users can share a name, so prefer the active record with a title and a manager, and ask if it is still ambiguous.
- **Date range:** the last 30 days by ClosedDate by default. If the user gives none, state the default used.
- **Statuses:** closed or resolved cases only.

## Steps

1. **Resolve the team.** Find the manager's User Id, then collect everyone below them: direct and indirect reports, walking `ManagerId` level by level until no new users appear. Only active users. List the people found and the count.
2. **Find the cases.** Cases owned by those users, in a closed or resolved status, closed inside the date range. Get Id, CaseNumber, Subject, Description, Status, Owner, ClosedDate, Product & Topic (`CaseRoutingTaxonomy__c`), Product Feature (`CaseReportingTaxonomy__c`) and IsEscalated. Skip cases with no activity beyond auto-generated messages.
3. **Check the log first.** For each case number, classify any existing entry:
   - **Human entry** (no AI tag): skip the case. Never overwrite or edit human content.
   - **AI entry** (carries the AI tag): skip, unless a human entry now exists (see step 9).
   - **No entry:** continue.
4. **Gather evidence per case.** Pull `EmailMessage`, `CaseComment` and feed posts. Then run the Swarm lookout.
   - **Swarm lookout** (all three channels, every case): <#C028BLR714Y> (support-swarm-gu-latam-amer), <#C027XT6D648> (support-swarm-security-and-activations) and <#C0BSLMLPWCW> (swarm-sales-cdmx). Search for (a) the case number, (b) 3-5 distinctive words from the Subject, and (c) the main points (error text, Product Feature, requested action, Org ID). Include bot messages. Search a window of about 60 days around the ClosedDate; if nothing is found, widen to 180 days.
   - Read the full thread of every relevant hit, not only the first message. Note the channel, who replied (full name), the resolution or next step, and what details the swarm asked for.
   - Record per case: **Swarm handled by** (channel and responder names, or "no swarm thread found") and the queries run. Never write "no swarm thread found" without having searched all three channels.
   - Use swarm answers as supporting context only. Do not quote customer-sensitive details. Cite the thread link in the review step, not in the log entry.
5. **Topic check.** Compare the case's Product & Topic with the topics the team works: Sales Gen Usage (all objects, data exports, how-to), Security and Activations, and Dev Support (optional). If it is none of these, add the label **Outside team topics** to the entry's Scope line with the Product & Topic found. If the topic is blank or unclear, do not label it.
6. **Decide if it teaches something.** Draft an individual entry when the case shows a non-obvious resolution, root cause, eligibility rule or scope call. Routine cases are not dropped: group them by theme (e.g. passkey/MFA, password resets, sandbox usernames, cancellations, data exports/archive, reports) into one combined historic entry per theme, listed as "(routine, grouped)".
7. **Draft the entry in the log's existing format.** Base it only on the evidence. If something cannot be determined, write "not determinable from case data". Never guess. For Product Feature use the case's `CaseReportingTaxonomy__c` value, or "not specified". Before drafting, read the 2-3 newest entries in the log and mirror their layout; never invent a new layout.
   - **Individual entry.** Heading `## ![](slack_date:CLOSED-DATE) Case NNNNNNNNN — Subject`, with the case number as plain text (no link in the heading). Bullets, in this order: **Worked by:** (full name of the case Owner); **Product & Topic:** value | **Product Feature:** value; **Scope:** IN SCOPE or OUT OF SCOPE (with a short why if OUT); **Rule / root cause:**; **Resolution:**; **Swarm handled by:** channel and responder, or "no swarm thread found"; **What worked with the customer:**. Then the AI-generated line and the Based on line.
   - **Grouped routine entry.** Heading `## ![](slack_date:MOST-RECENT-CLOSED-DATE) Theme (routine, grouped)`. Bullets, in this order: **Worked by:** (each owner with their case numbers in parentheses); **Cases closed:** first date to last date; **Product Feature(s):**; **Scope:** IN SCOPE, routine; **Swarm handled by:** (typical channel and responder, if any); **Summary:** (the common issue and the standard handling); **Cases:** a nested bullet per case as a Salesforce link with `(closed YYYY-MM-DD)`. Then the AI-generated line and the Based on line (Low confidence if only subjects and taxonomy were reviewed).
   - Separate entries with a `---` divider, newest ClosedDate on top.
   - Add these two lines to every drafted entry:
     - `:robot_face: AI-generated from case data (emails, comments, posts, swarm thread). Not provided by a human, so details may be missing.`
     - `Based on: [list the sources actually found, e.g. 6 emails, 3 comments, swarm thread found/not found]. Confidence: High / Medium / Low.`
   - **"What worked with the customer" rule:** one or two sentences, in plain language, on the reply approach that moved the case forward (for example the one detail that unblocked it, the article or step that resolved it, or the tone that calmed an escalated customer). It must come from what the case emails actually show. Never include customer names, internal W-numbers or other customers' details. If the emails do not show it, write "not determinable from case data". This field feeds the message templates of case-first-contact.
   - **Date rule:** the heading date must be the case's ClosedDate (YYYY-MM-DD), never the date the skill is run. For a grouped entry, use the most recent ClosedDate and list the range.
8. **Review.** Show the proposed entries, the swarm findings per case (with thread links and queries run), the skipped cases and the cases already covered. The user decides what to add and pastes it into the log, newest ClosedDate on top. Do not write anything to Slack.
9. **Upgrade AI entries when a human entry arrives.** On each run, also check the AI-tagged entries already in the log. If a human has since added an entry for the same case number (for example through case-first-contact), recommend keeping the human entry and removing the duplicate AI entry, so no AI tag remains on that case. If a human edited the AI entry itself, recommend removing the AI tag and keeping the human text. Report these suggestions; the user applies them.

## Output

- **Team:** [Manager], [N] people: [names] | Range: [dates] | Cases found: [n] | Already logged: [n] | Skipped (routine): [n] | Proposed: [n]
- **Proposed entries**, for each case: `Case [CaseNumber] — [Subject] ([link])`, then Worked by | Product & Topic | Product Feature; Scope call (+ "Outside team topics" if the topic check says so); Rule or root cause; Resolution; Swarm handled by (+ "Searched: 3 channels, window, N results"); What worked with the customer; the AI tag and Based on / Confidence lines.
- **Skipped (routine):** [case number — reason]. **AI entries to upgrade or remove:** [case number — what changed].
- Then ask: "Which of these should I prepare to paste?"

## Key Principles

- Never present AI-drafted content as human-provided. The tag is required on every entry.
- Never edit or delete human-written entries.
- Never invent data. Missing information is "not determinable from case data".
- Link every case and article with its Salesforce URL.
- Treat the swarm channels as supporting context only. Do not quote private or sensitive customer details in the log.
- Always search all three swarm channels before writing "no swarm thread found".
- Keep the entry layout compatible with case-first-contact, which reads these entries.
- If the manager or team cannot be resolved, say so and stop rather than guessing.
