## Linear ticket

Resolves TRA-
Link:

## Type of change

- [ ] feat — new feature
- [ ] fix — bug fix
- [ ] chore / refactor
- [ ] migration — DB schema change
- [ ] ci — workflow / tooling

## Summary

<!-- 2-3 sentences: what changed and why. For features, include the end-to-end flow. -->

## Technical decisions

<!-- Specific implementation choices a reviewer would question.
     Format: "We chose X over Y because Z." Delete this section if nothing warrants it. -->

## Notes

<!-- Operational caveats, rollback levers, known limitations, or follow-up work.
     e.g. "This feature flag is a rollback lever — flip to false to revert without a redeploy."
     Delete if nothing to add. -->

## Checklist

- [ ] Migration added (if schema changed)
- [ ] RLS policy added/updated (if new tenant table)
- [ ] Feature flag needed? If yes, added to `Organisation.configuration.featureFlags`
- [ ] New env vars added to all environments (staging, production, preview)
- [ ] Backwards compatible? Existing API callers and data unaffected
- [ ] Breaking change? If yes, document it here
- [ ] CODEOWNERS updated (if new directory paths added)

---

<!--
  Large PR? (>30 changed files) Fill in the reviewer guide below.
  It's the highest-ROI thing you can write. Delete if not needed.
-->
<details>
<summary>Reviewer guide</summary>

**Real review surface:** X of Y changed files need attention. The rest are [DTOs / generated / proxy routes / planning docs — describe].

**Read these first (in order):**

| File              | Why it matters            |
| ----------------- | ------------------------- |
| `path/to/file.ts` | Core logic / highest risk |

**Skip outright:**

| Path(s) | Reason                                    |
| ------- | ----------------------------------------- |
| `path/` | Generated / reindent / planning artifacts |

**Highest-risk areas:** <!-- Where to focus scrutiny and why -->

</details>
