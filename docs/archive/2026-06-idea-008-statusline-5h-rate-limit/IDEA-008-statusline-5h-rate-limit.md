---
id: 008
title: Add 5-hour rate-limit segment to the status line
status: complete      # idea | in-progress | complete | superseded
priority: medium      # high | medium | low
supersedes: []        # list of IDEA ids this replaces, or []
superseded_by: null
depends_on: []        # list of IDEA ids required before starting, or []
related: []           # list of IDEA ids that share context, or []
created: 2026-06-01
completed: 2026-06-01
# Sprint-auto eligibility gates — both must be `true` with explicit reasoning
# before sprint-auto can run this idea unattended overnight.
# Default to `false` at capture; upgrade in `/plan` once the unknowns are nailed down.
auto_safe: false                                            # true | false
auto_safe_reason: "Mostly additive single-file edit, but the exact statusLine JSON field path for the 5h window is unverified, and there are UX choices (label text, color-tier thresholds, segment placement relative to the existing 7d segment) that sprint-auto would otherwise decide blindly."
sensitive_paths_cleared: true                              # true | false
sensitive_paths_cleared_reason: "Scope is limited to scripts/statusline-command.sh — a pure display/rendering script with no auth, permission, schema, infra, or secrets surface."
---

# IDEA-008: Add 5-hour rate-limit segment to the status line

**Status**: ✅ Complete (2026-06-01)
**Priority**: Medium

**Problem** (or opportunity): `scripts/statusline-command.sh` surfaces the 7-day rolling rate limit (segment 4, `7d:NN%`, from `rate_limits.seven_day.used_percentage`) but shows nothing for the 5-hour rolling window. The 5h limit is the one that gates a working session in the short term, so it's the more actionable budget to keep an eye on — yet it's currently invisible in the status line.

**Proposal** (or idea): Add a `5h:NN%` segment next to the existing `7d:` segment, fed from the 5-hour rate-limit field in the statusLine JSON input (likely `rate_limits.five_hour.used_percentage` — exact key TBD, verify against a live `--debug` dump of the status line stdin). Reuse the existing dim/yellow-bold/red-bold tier coloring from the ctx and 7d segments (<50 / 50–79 / ≥80) so the three percentage meters read consistently. Only render when the field is present (same `if [ -n … ]` guard pattern as the other segments), so older Claude Code builds that don't emit it degrade gracefully.

**Why now**:
- The status-line script already exists and is wired up (PR #136 / `statusline-command.sh`), so this is a small, self-contained extension to working code rather than net-new plumbing.
- The 5h window is the limit users actually bump into mid-session; surfacing it alongside the 7d figure closes an obvious gap in the rate-limit visibility.

**Non-goals**:
- No restructuring of the existing segments or color scheme — purely additive.
- No reset-time / countdown display (just the used-percentage meter, matching the 7d segment's level of detail).
- No changes to the symlink installer or jq-fallback path.

**Related**: None. Touches the same file shipped by the recent statusline work (commit 6f5d731) but is a standalone enhancement.
