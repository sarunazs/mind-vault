# Dependency selection criteria — N-1, no community forks, ecosystem first

When `/plan` recommends a framework / package / CMS / admin panel, apply these criteria **in order**. Load this reference at step 4 of `/plan` (the "Key Technical Decisions" section) any time the plan involves picking new dependencies.

## The criteria, ranked

1. **Ecosystem familiarity / community size.** If the user has "never heard of it", that's a strong negative signal. Recommend mainstream packages by default; surface niche-but-technically-better candidates with extra caveats. Long-term solo maintenance benefits more from "tutorials + Stack Overflow + colleagues who've shipped this" than from "marginally better DX".
2. **True OSS, no paid tier.** Preferred over open-core packages that gate features behind a license, even when the free tier covers current needs. The paid-tier threshold is future friction the user wants to avoid hitting.
3. **N-1 version rule, verified against current latest.** When the latest major release of a framework is <12 months old AND requires community-fork workarounds (un-released official plugins, deprecated well-known plugins, ecosystem still catching up), **drop to the previous major**. "N-1" must be derived from the **actual current latest**, not from intuition.

   The verification: run the canonical "what versions exist" command for the package manager:

   ```bash
   composer show -a <package> | grep ^versions       # PHP / Composer
   npm view <package> versions --json | tail -20      # Node / npm
   pip index versions <package>                       # Python / pip
   gem list -r ^<package>$ --all                      # Ruby / gem
   ```

   Pick the latest stable from the output, then drop to its previous major for the N-1 default. If the user said "use the version one below", that means N-1 from **today's latest**, not from "what feels like the standard version".

4. **No community forks of official packages.** If staying on the latest major forces a `community-org/X` fork (where `X` is the official `<vendor-org>/X` plugin on N-1), the latest major is the wrong choice. Use the version where the package lives under the official org's namespace.

5. **GitHub stars + active plugin ecosystem.** Rough proxy for "can I find tutorials, hire help, Stack-Overflow my way out". Treat order-of-magnitude differences as decisive; 10k vs 30k stars is noise, 3k vs 30k is signal.

6. **Speed-to-MVP.** Real, but **never a tiebreaker against the above.** A package that ships faster but requires a community fork or recently-deprecated plugin costs more in the long tail than the speed gained at the start.

## Apply the rule across the stack, not just the framework

When a framework drops a major, **carry its CSS / build / component-library partners with it**. Don't mix N-1 of the framework with N of its peers — the ecosystem snapshot is what's stable, not any single package in isolation.

Examples of the bundle-level N-1 (or N-2 if the framework's latest is <6 months old):

- Filament v3 (admin) ⇄ Tailwind v3 (Filament v3 ships v3 natively) ⇄ Flowbite v2 (Tailwind-v3 line) ⇄ Spatie packages compatible with Laravel 12 (Filament v3's framework constraint).
- Don't mix Filament v3 admin with Tailwind v4 public — they share a build chain.
- Don't mix Laravel 12 with packages whose latest only supports Laravel 13 — `composer require --with-all-dependencies` will downgrade something unexpected.

## Anti-patterns

- ❌ **"Use the latest" by default**, then handle pain as it appears. Cheaper to make the right call up front; the cost of changing framework majors mid-project is far more than the cost of one extra minute checking versions.
- ❌ **"N-1 means the version I happen to remember"** — without verifying against the package's actual version list. If the framework released a new major last month, "the version I remember" is now N-2 and you've drifted further behind than the user asked.
- ❌ **Swapping to a community fork** without flagging it. The fork has different maintenance, different release cadence, different security-disclosure flow. If the official package isn't ready, the framework isn't ready.
- ❌ **Treating speed-to-MVP as a tiebreaker** when the user signalled long-term maintenance preference. Solo developers maintaining projects for years discount future friction at much lower rates than VC-funded teams optimizing for launch velocity.

## When to override

- **Brownfield codebase already on N**: don't recommend a downgrade. Apply N-1 only at greenfield decision points.
- **N-1 is end-of-life**: if the previous major is past its support window and the latest major has a clean ecosystem, take N.
- **Security advisory on N-1**: take N (or stay on N-1 with a documented patch level).
- **User explicitly asks for cutting edge**: honour their stated preference; flag the trade-off but don't override.

## Apply during /plan's "Key Technical Decisions" section

For each recommended package in the plan:

1. Run the version-listing command for the package manager. Pin the resolved version in the plan.
2. State explicitly "N-1: vX.Y per `composer show -a <pkg>` 2026-MM-DD" so the reviewer can see the version was verified, not guessed.
3. Confirm the package is in the official organisation's namespace (not a community fork).
4. Flag open-core packages with their paid-tier trigger condition ("Pro license unlocks at $X / kicks in when …").
5. If a peer package (Tailwind, build tool, component library) needs to track the framework's major, lock it in the same table.

The N-1 table from a real plan, redacted for generality:

```text
| Layer            | Choice              | Version             | Why                            |
|------------------|---------------------|---------------------|--------------------------------|
| Framework        | <framework> N-1     | ^X.Y                | N-1 per `<verify-command>`     |
| Admin panel      | <admin> N-1         | ^X.0                | Latest is <N months>, requires |
|                  |                     |                     | community fork — drop to N-1   |
| CSS              | <css-fw> N-1        | ^X                  | Tracks <admin>'s native build  |
| Component lib    | <component-lib> N-1 | ^X                  | Tracks <css-fw>'s major        |
| Translatable     | <official-plugin>   | ^X.0                | OFFICIAL, not the fork         |
```

## When NOT to use this

- The plan doesn't introduce new dependencies (refactor / docs-only / bug fix on a frozen stack).
- The user explicitly opted into N or HEAD ("I want to use the latest, I understand the risks").
- The project is a research prototype with no maintenance horizon.

---

**Last Updated**: 2026-05-22 (compounded after a session where the agent recommended a framework's latest major in the plan, the user pushed back with "use the version one below", and the agent picked the next-down version — which turned out to be N-2 because the latest had just been released. The verify-against-current-latest step was the missing discipline.)
