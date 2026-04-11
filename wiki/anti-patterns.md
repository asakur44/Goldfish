---
type: anti-patterns
triggers: [rejected, failed, abandoned, don't use, anti-pattern, mistake]
---

# Anti-Patterns

Approaches we tried and rejected. Check here before proposing a solution — if it's listed below, explain why the original rejection no longer applies before re-suggesting it.

## Template

Each entry should include:
- **What was tried**
- **Why it failed** (specific, not vague)
- **When / context** (constraints at the time — things may have changed)
- **Source** (conversation, document, or incident reference)

---

## Rejected Approaches

### [Example] Roll-our-own JWT refresh logic
- **Tried:** 2025-11
- **Failed because:** Refresh token rotation logic was error-prone. Race conditions when multiple tabs refresh simultaneously caused token invalidation cascades.
- **Context:** At the time, we had no shared token store. With a Redis-backed store this might be viable, but the complexity isn't justified while Auth libraries handle it.
- **Source:** Discussion in 2025-11 architecture session

<!-- 
Add your own rejected approaches below. Delete the example above.
Format: what, why it failed, when/context, source.
-->
