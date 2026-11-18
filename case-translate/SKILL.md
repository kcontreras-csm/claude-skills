---
name: Case Translate
version: 1.0.0
description: Translate a Salesforce support case (Subject, Description and the last 20 comments) into English for a language-support hand-off, given a case number and the source language. Use when the user says "translate case", "translate to English", "traducir caso" or "case translation" along with a case number (e.g. /case-translate 476196596 Portuguese). Translation only: no troubleshooting, no scheduling, no questions to the customer.
---

Credits: based on the "Translate Support Case to English" skill by Abraham Zaid Aguilar (Slackbot).

Translate the case in the arguments into English and prepare the hand-off to the parent-case agent. Draft only: never post comments or change the case.

## Tools (Claude Code)

- Load via ToolSearch `select:mcp__orgcs__soqlQuery,mcp__orgcs__getUserInfo` (OrgCS and H&T org). The current user's name, role and time zone come from `getUserInfo`. Never use a name from an earlier run or from an example.
- If SOQL is blocked by the security policy ("The operation you requested isn't allowed due to a security policy in your organization"), use the fallback in the case-first-contact skill (SOSL for the case, `Task` records for emails) and mark what could not be read.

## Steps

1. **Scope.** Inputs: the case number and the source language. Ask for either if missing. If the number is a **Parent** case and not a **Child** language case, ask the user which one to translate. Translations should generally target the Child case content, per the language-support process.
2. **Gather.** Look up the case by number: `SELECT Id, CaseNumber, Subject, Description, Status, ParentId, Parent.CaseNumber, Parent.Owner.Name FROM Case WHERE CaseNumber = '<number>' LIMIT 1`. If it is a Parent, also list its children (`WHERE ParentId = '<Id>'`) so the user can choose. Then the comments, newest first, capped at 20: `SELECT CommentBody, IsPublished, CreatedBy.Name, CreatedDate FROM CaseComment WHERE ParentId = '<Id>' ORDER BY CreatedDate DESC LIMIT 20`. If the case number is not found, say so. Never guess a similar number.
3. **Translate** the Subject, the Description and every non-English comment into English.
   - Keep technical terms, product names, error codes and IDs exactly as written. Do not localize or "correct" them.
   - Keep it plain and literal. Flag any ambiguous or idiomatic phrase instead of guessing its meaning, e.g. `[ambiguous: reading A / reading B]`.
   - Comments already in English stay as they are, labeled "(original, no translation needed)".
   - Illegible or missing text is marked `[unclear in source]`. Never fabricate a translation.
4. **Output**, in this order:
   - **Case Summary (English):** Subject and Description, translated, plus Status and the Parent/Child relationship.
   - **Comment Timeline (English), newest first:** each entry as `[date] [original language] → English: "..."`.
   - **Internal-comment-ready translation** (only for a Child language case): plain text, phrased as if speaking to the customer, with a placeholder `@[Parent-case agent]`. Translators only translate: they do not ask for information, schedule or troubleshoot.
   - **Hand-off message to the parent-case agent.** Always include this standard message, per the AMER language-support process. Use the current user as the translator and ask for their working hours and time zone if unknown; otherwise leave a placeholder:

     Hi @[Parent-case agent], my name is [Name of the person running the skill] and I will assist you with this translation. I have created the child case: [Child case number]. My calendar is up to date and my working hours are [working hours + time zone]. Let me know once you take the child case so I can take the parent case and I'll be pending your response.

   - **Footer:** case number, language translated from, and "[no comments found]" if the case has none.

## Key Principles

- Never fabricate a translation for illegible or missing text. Mark it `[unclear in source]`.
- Language agents are translators only. The output must not add troubleshooting, scheduling or requests for information that were not in the original case content.
- Preserve case numbers, IDs and technical terms untouched.
- Never hard-code a person's name in the hand-off message.
