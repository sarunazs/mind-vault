# IDEA-008 — Add 5-hour rate-limit segment to the status line

**Status**: ✅ Complete
**Completed**: 2026-06-01
**PR**: _(pending — see index)_
**Related**: builds directly on the status-line script shipped in commit `6f5d731` (PR #136).

## What shipped

A new `5h:NN%` segment in `scripts/statusline-command.sh`, fed from `rate_limits.five_hour.used_percentage`, rendered immediately **before** the existing `7d:` segment so the two rate-limit meters read in escalating-window order (`ctx → tokens → 5h → 7d → effort → vim`).

Segment order rendered:

```
📌 topic │ ctx:NN% │ ⬆in/⬇out ↺cache │ 5h:NN% │ 7d:NN% │ 🧠 effort │ -- MODE --
```

## Implementation notes

- **Field path** (confirmed against the [official statusLine docs](https://code.claude.com/docs/en/statusline)): `rate_limits.five_hour.used_percentage`. Mirrors the existing `rate_limits.seven_day.used_percentage`. A sibling `resets_at` (unix epoch) is also available but intentionally not surfaced — non-goal (no countdown display).
- **Tier colors** reuse the exact thresholds already used by `ctx` and `7d`: dim (<50) / yellow-bold (50–79) / red-bold (≥80) — three percentage meters stay visually consistent.
- **Graceful degradation**: `five_hour` may be independently absent (non-subscriber, or before the session's first API response). The same `if [ -n "$five_hour" ]` guard as every other segment means the meter simply doesn't render — verified that 5h-absent, 7d-present and fully-absent `rate_limits` cases both render cleanly with no errors.
- **jq extraction**: `five_hour` was inserted adjacent to `seven_day` in the single-pass jq invocation (kept the one-process-per-render hot-path optimisation intact); downstream `_sl_fields` index assignments shifted by one accordingly.

## Verification

`bash -n` clean. Four functional cases exercised: both windows present (mid tiers), 5h ≥80 red / 7d <50 dim, 5h absent (degrades to 7d-only), and `rate_limits` entirely absent (old build → neither segment, no error).

## Why now

- The status-line script already existed and was wired up (PR #136), so this was a small additive extension to working code rather than net-new plumbing.
- The 5-hour window is the limit users actually bump into mid-session; surfacing it alongside the 7-day figure closed an obvious visibility gap.

## Non-goals (held)

- No restructuring of existing segments or the color scheme — purely additive.
- No reset-time / countdown display (`resets_at` left unused).
- No changes to the symlink installer or the jq-missing fallback path.
