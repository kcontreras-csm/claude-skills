---
name: Case Person Snapshot
version: 1.0.0
description: Build a manager-ready snapshot of how ONE person is doing with their cases in OrgCS, given a name. Covers open and closed cases, responsiveness, escalations, workload, a comparison with the previous 30 days and with peers under the same manager. Use when the user says "person snapshot", "how is [name] doing with their cases", "snapshot de [nombre]", "revisar casos de [nombre]" or wants to see whether a teammate needs help or prepare a 1:1. Read-only.
---

Credits: based on the "OrgCS Person Case Snapshot" skill by Abraham Zaid Aguilar (Slackbot).

Build a snapshot for ONE person. It is performance and workload information about a named colleague: use it to offer help or in a management review, and keep the output private. Read-only: never change a case or send anything.

## Tools (Claude Code)

- Load via ToolSearch `select:mcp__orgcs__soqlQuery,mcp__orgcs__getUserInfo,mcp__orgcs__getObjectSchema` (OrgCS and H&T org, org id 00D30000000XsfGEAS).
- If CloudSuccessCaseMCP / CSMOrgCSMCP tools are available and expose case lookup, prefer them; otherwise use plain queries.
- Any metric the org does not expose is shown as "not available" or labeled as a proxy. Never estimate.
- If SOQL is blocked by the security policy, use the fallback in the case-first-contact skill and mark what could not be read.

## Steps

1. **Scope.** Input: a person's name (ask if missing). Window: the last 30 days by default (the user may change it). The comparison window is the 30 days immediately before it. Open cases are always a current snapshot, whatever the window. This covers one individual: the team view is context for that person, not a team report.
2. **Resolve the person.** Look up the User by name. No match: write "[no user found for 'name']" and stop. Several matches: list up to 5 (name, title, email) and ask which one.
3. **Gather** (bulk queries only, no per-case searches):
   - **The person's cases.** One query for cases owned by the person that are open now (max 25, ordered by priority, then oldest first) or closed in the last 60 days (current plus comparison window, max 200). Fields: CaseNumber, Subject, Status, Priority, IsEscalated, IsClosed, CreatedDate, ClosedDate, LastModifiedDate, Origin, Product & Topic (`CaseRoutingTaxonomy__c`), Product Feature (`CaseReportingTaxonomy__c`), Account name, and the first lines of the Description. Add any SLA, milestone, entitlement or first-response field the org exposes on Case (check with `getObjectSchema`). If a cap is hit, state the total and which part was truncated.
   - **First response.** If the org exposes a first-response field on Case, use it. Otherwise run one bulk query of outbound `EmailMessage` records (`Incoming = false`) for the cases created in the 60-day range and use the earliest outbound email per case as a proxy. If neither works, show "not available".
   - **Team context.** Find the person's manager and active peers (same `ManagerId`, max 25). Run one aggregate query over the peers' cases for the same 60-day range, grouped by owner: open count, closed count, escalated count and average days to close. If the manager or peers are not found, skip the Team section and say so in the footer.
   - **CSAT.** One query for survey responses tied to the person's closed cases, only if the org exposes survey or CSAT data. Otherwise show "CSAT: not available" and do not infer a score.
4. **Assess** from the fetched data only. Compute each metric for the current window and the previous 30 days, and show the change (up, down or flat, with the delta):
   - **Backlog:** open now, opened in window, closed in window, net change.
   - **SLA risk** (open cases): if the org exposes a milestone or target date, flag Breached / At risk (target within 24 h) / On track. Otherwise derive a proxy from Priority, IsEscalated and case age, labeled "age-based proxy, not an official SLA". Never invent a target date.
   - **Stale:** open cases with 3+ days since the last update.
   - **Responsiveness:** average and median time to first response, and average and median days to close, for cases closed in the window. State the source (official field or proxy).
   - **Quality:** escalation rate and closure status mix for cases closed in the window.
   - **Workload mix:** top Product & Topic areas, and the number of cases with no Product Feature set.
   - **CSAT:** average score and response count, if data was found.
   - **Team position:** rank the person among peers on open count, closed count and average days to close. Flag if the person is well behind the peer median. This reflects workload and process only; case complexity, mix and availability are not captured.
   - A one-line summary per open case from Subject and Description (Subject only if the Description is blank).

## Output

**Manager Snapshot: [Person name] ([title]), last [N] days vs previous [N]**

- **Headline:** [open now] open | [closed in window] closed | [escalated] escalated | [breached or at risk] SLA risk | [stale] stale. Needs attention first: up to 3 case numbers with the reason, or "none flagged".
- **Performance vs previous period** (table: Metric | Current | Previous | Change): closed cases, opened cases, avg / median first response, avg / median days to close, escalation rate, CSAT (responses). Use "not available" where the data is missing.
- **Team position:** one or two sentences citing the numbers, or "not available: manager or peers not found". Table: Person (this person in bold) and Peer median, with Open, Closed (window), Avg days to close, Escalated.
- **Workload mix:** top Product & Topic areas with counts | N cases with no Product Feature | status mix of closed cases.
- **Manager takeaways:** up to 3 bullets drawn only from the numbers above, framed as workload and process, not as judgments about the person.
- **Open cases** (table: Case | Subject / Summary | Status | Priority (⚠ if escalated) | SLA | Age / last update ("stale" if 3+ days) | Product & Topic).
- **Footer:** open cases found vs shown; closed cases found vs shown; first-response source (official field, outbound-email proxy or not available); SLA source (official field or proxy); CSAT source or "not available"; team context source or why it was skipped.

## Key Principles

- Missing data shows "not available" or "not specified". Proxies are always labeled as proxies.
- Link every case with its Salesforce URL: `https://orgcs.lightning.force.com/lightning/r/Case/<Id>/view`.
- Team comparison is context about workload and process, not a verdict on the person. Say so when flagging that someone is behind.
- Keep it a summary: no drafts, no similar-case or knowledge searches. Offer `/case-first-contact` for a deeper dive on any specific case.
