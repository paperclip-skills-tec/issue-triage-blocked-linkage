---
name: issue-triage-blocked-linkage
description: Enforces the blocked metadata contract for Paperclip issues. Invoke before setting any issue to `blocked` status — ensures `blockedByIssueIds` is set for internal blockers, or a structured external-blocker comment (owner, expected resolution date, unblock condition) is posted for external blockers. Also use when triaging an existing blocked issue that may be missing linkage. Without proper linkage, automated unblock wakes cannot fire and issues stay stuck indefinitely.
---

# issue-triage-blocked-linkage

Blocked issues without machine-actionable metadata become permanently stuck: the automated `issue_blockers_resolved` wake never fires, and no agent or human knows when or how to resume work.

This skill is a pre-flight checklist. Run it before every blocked transition. It takes about 30 seconds and prevents issues from disappearing into limbo.

## When to invoke

- You are about to `PATCH /api/issues/{id}` with `status: "blocked"`
- You are triaging an existing issue that is already in `blocked` status
- You see free-text "blocked by X" in a comment but no structured linkage

## Step 1 — Classify the blocker

Ask yourself: **is the blocker another Paperclip issue, or something external?**

| Blocker type | Example | Required action |
|---|---|---|
| **Internal** — another Paperclip issue | "waiting on TEC-204 to be merged" | Set `blockedByIssueIds` |
| **External** — outside Paperclip | "waiting on vendor API key", "pending legal sign-off" | Post structured external-blocker comment |

If you're unsure, lean toward internal if you can find or create a tracking issue. External is for things genuinely outside the system.

## Step 2a — Internal blockers: set blockedByIssueIds

Resolve the blocking issue's UUID (from its identifier, e.g. `TEC-204`):

```bash
GET /api/companies/{companyId}/issues?q=TEC-204
```

Then patch the blocked issue, setting both status and the blocker array in one call:

```bash
PATCH /api/issues/{issueId}
{
  "status": "blocked",
  "blockedByIssueIds": ["<blocking-issue-uuid>"],
  "comment": "Blocked on [TEC-204](/TEC/issues/TEC-204) — <one sentence why>."
}
```

The `blockedByIssueIds` array **replaces** the current set on each update. To add to existing blockers, fetch the current list first and merge. To clear all blockers, send `[]`.

**What happens next:** Paperclip monitors the blocking issues. When all of them reach `done`, it fires `issue_blockers_resolved` on the dependent issue and wakes the assignee automatically. No polling needed.

## Step 2b — External blockers: post structured comment

When the blocker lives outside Paperclip (third-party, team, legal, infrastructure), free-text is not enough. Post a comment with all four fields:

```
## Blocked — external dependency

- **Blocker**: [what system or team is blocking this]
- **Owner**: [named person or team responsible for unblocking — be specific]
- **Expected resolution**: [absolute date, e.g. 2026-06-01 — not "next week"]
- **Unblock condition**: [the specific deliverable or state that clears this block]
```

Then patch status to `blocked` (without `blockedByIssueIds` since there is no internal issue):

```bash
PATCH /api/issues/{issueId}
{
  "status": "blocked",
  "comment": "## Blocked — external dependency\n\n- **Blocker**: ...\n- **Owner**: ...\n- **Expected resolution**: ...\n- **Unblock condition**: ..."
}
```

**Why all four fields?** Owner tells future agents who to ping. Expected resolution date tells them when to check back. Unblock condition removes ambiguity about what "resolved" means. Missing any of these forces manual investigation and delays resumption.

## Step 3 — Verify before exiting

After patching, confirm the issue shows the correct state:

```bash
GET /api/issues/{issueId}
```

Check:
- `status` is `blocked`
- `blockedBy` array is populated (for internal blockers), OR the last comment contains all four structured fields (for external blockers)

If either check fails, fix it before moving on.

## Triaging pre-existing blocked issues

When you encounter a blocked issue during triage (e.g., running issue-triage or a hygiene scan):

1. **Has `blockedByIssueIds`?** → Compliant. Check if blocking issues are `done` — if so, patch status back to `in_progress` and wake the assignee.
2. **Has a structured external-blocker comment with all four fields?** → Compliant. Check if the expected resolution date has passed — if so, ping the owner.
3. **Has only free-text "blocked by X"?** → Non-compliant. Post a comment requesting the assignee add `blockedByIssueIds` or the four-field external-blocker block. Flag in triage summary.
4. **No comment at all?** → Escalate in triage summary under "Issues requiring human decision."

## Reference

This skill enforces the rule described in [TEC-1087](/TEC/issues/TEC-1087) and the **Blocked status requires explicit linkage** guardrail in `AGENTS.md`.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
