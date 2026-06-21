# PR #6405 — autoreview findings + handoff

**Branch:** `feat-ios-paired-mac-backup` (worktree `worktrees/feat-ios-paired-mac-backup`)
**HEAD:** `b82c6e25e` (Make presence production-ready)
**Scope:** started as "iOS: don't lose saved hosts/IPs on upgrade (paired-Mac backup + restore)";
grew to multi-Mac aggregation + per-Mac customizations (name/color/icon) + Stack **teams**
(team-scoped data + team picker in Settings) + **presence made production-ready** (follows the
mobile toggle; prod URL on release) + a **build-channel** badge on the Computers screen (DEV·tag /
Nightly / RC / Stable). 74 files, ~+8578/−1920, ~40 commits.

> This file is a scratch handoff, not meant to be committed. `git rm --cached AUTOREVIEW-HANDOFF.md`
> (or delete it) before merge.

## Review state

- **Structured Codex correctness review: CLEAN** (the wrapper only runs `cmux-policy-check` after a
  clean structured pass). All P0/P1 correctness findings from ~10+ prior rounds are fixed and
  committed.
- **`cmux-policy-check` (Aziz policy): 63 P2 findings remain.** These are the merge-gate blockers for
  a clean autoreview. Re-capture any time with:
  ```bash
  /Users/lawrence/fun/cmuxterm-hq/skills/autoreview/scripts/cmux-policy-check --mode branch --base origin/main
  ```

## The 63 findings, by category

| Category | Count | What it wants |
| --- | --- | --- |
| Aziz documentation | 39 | Public package symbols need `///` Swift-DocC docs |
| Aziz file-organization | 14 | One major type per file; split added types into `TypeName.swift` |
| Aziz package-design | 7 | No-case namespace enums with static members → injectable structs / file-scope helpers; pure helpers → file-scope private funcs (not `private static`) |
| Aziz testability | 2 | Inject `defaults`/`FileManager`/paths seams (`MobileShellComposite.setForegroundWorkspaceState` area) |
| Aziz concurrency | 1 | `DeviceTreeView.swift:147` `Task.sleep` (the Computers poll timer) — runtime waits should use signals, not sleep |

### By file (top offenders)
- `PairedMacBackupClient.swift` — 15 (mostly doc on `PairedMacBackupRecord` fields + the `PairedMacBackingUp` protocol; one file-org for the `PairedMacBackupOp` enum sharing the file)
- `BackingUpPairedMacStore.swift` — 10 (doc on the public actor + its methods)
- `MobileShellComposite.swift` — 7 (doc on `currentTeamDidChange` etc.; file-org for `WorkspaceMutationTarget`/`SecondaryMacSubscription` nested types; testability seams; a `private static` pure helper)
- `PairedMacRestore.swift` — 5 (doc + file-org for `RestoreOutcome`)
- `MobileWorkspaceListFilter.swift` / `MacWorkspaceState.swift` — 4 each (doc + file-org for `MobileWorkspaceAggregation`)
- `MobilePairedMacBackup.swift` — 4 (doc)
- `MobilePairedMacStore.swift` — 3 (doc)
- `MachineAvatarColors.swift` / `DeviceTreeView.swift` — 2 each
- `MachineAvatarPalette.swift`, `MacBuildChannel.swift`, `PresenceMap.swift`, `MacComputerRow.swift`, `WorkspaceListFilterControls.swift`, `MacPairedMacBackupPublisher.swift`, `PairedMacBackupTests.swift` — 1 each

Full raw list (file|severity|rule): `/tmp/policy-findings.txt` on this machine. Regenerate with the
command above.

## Triage guidance (recommended dispositions)

- **Documentation (39): FIX.** Mechanical, clear value. Add `///` to each flagged public symbol. The
  bulk are simple value-type fields (`PairedMacBackupRecord`, `MacWorkspaceState`,
  `MobileWorkspaceListFilter`) and public methods/protocols. Low risk.
- **File-organization (14): FIX the easy ones, REJECT-with-evidence the tightly-coupled ones.**
  Splitting `PairedMacBackupOp` out of `PairedMacBackupClient.swift`, `RestoreOutcome` out of
  `PairedMacRestore.swift`, etc. is straightforward. For closed sets of tightly-coupled DTOs/fixtures
  that splitting would *reduce* clarity, reject per the cmux-policy triage rule (name the coupling).
- **Namespace-enum → struct (part of package-design, ~4): REJECT-with-evidence.**
  `MobileWorkspaceAggregation`, `MachineAvatarColors`, `MachineAvatarPalette` are **pure, stateless
  function namespaces** (no instance state, same-input→same-output, already unit-tested as pure
  functions). Converting to injectable structs adds ceremony without improving testability. This is a
  defensible conscious exception — state the invariant in a brief inline comment.
- **`private static` pure helpers → file-scope (package-design): FIX (cheap)** where it doesn't churn
  call sites.
- **Testability seams (2): FIX or REJECT-with-evidence.** `setForegroundWorkspaceState` is MainActor
  store-internal; injecting `FileManager` there is not meaningful (it touches no FS). If the seam is
  genuinely not testable-improving, reject with that evidence.
- **Concurrency / `Task.sleep` (DeviceTreeView:147): DECIDE.** This is the **Computers-screen poll
  loop the user explicitly asked for** ("poll more often when the sheet is expanded"). It's bounded
  (10s), sequential (awaits each refresh before the next sleep), and cancelled on dismiss (the
  `.task` is torn down). Either (a) reject-with-evidence (bounded UI refresh, not a synchronization
  substitute, cancelled on dismiss) with an inline comment, or (b) convert to a clock-based
  cancellable timer sequence if you want strict policy conformance.

## Merge blockers (do NOT auto-merge)

1. **Dogfood approval (CLAUDE.md).** This is an iOS app/runtime PR. The user has been actively
   reporting issues this session (drawer UX, presence "unknown"/"feels wrong") that were fixed in the
   *latest* commits; those latest builds are **not yet confirmed-good by the user**. CI/review/
   self-verification do NOT replace dogfood approval. Need an explicit "looks good to merge."
2. **The 63 policy findings** must be fixed or consciously accepted by the PR owner.
3. **Build:** the mac `teams` build is COMPLETE (`com.cmuxterm.app.debug.teams`, deeplink `http://127.0.0.1:17320/teams` → 200, running). iOS `teams` was also built this session. A fresh tagged reload is still needed at merge-readiness if the tree changed.
4. **Production presence note:** on merge, `.github/workflows/presence.yml` deploys the updated worker
   (bundleId/build-channel, customization-merge, tombstone cap) to **prod** (`presence.cmux.dev`),
   completing the production-presence path. The dev worker (`cmux-presence-dev`) is already deployed
   with these changes.

## Strong recommendation

This PR has outgrown its title. Consider splitting the newer additions (**teams**, **presence-prod**,
**build-channel**) into separate follow-up PRs so each is independently reviewable — the policy-finding
count and the review surface are both symptoms of the breadth. The original "backup + multi-Mac" core
is solid and correctness-clean.

## Dogfood environment (for verifying on device)

- Tags built this session: **`teams`** (current dogfood; unique, isolated) and **`dog`** (the stable
  shared dogfood tag). Both mac + iOS were built on `teams`. iOS device: `E4058DA9-F4C7-52DD-951D-0354061B8E89`.
- A fresh `teams` bundle starts **logged out**; dev sign-in/pairing need the local Next.js dev server.
- **Presence on the `teams` dogfood** needs the `teams` *Mac* build to have **mobile pairing enabled**
  + signed into the **same Stack team** as the phone (both DEBUG builds point at the **dev** worker).

---

## HANDOFF PROMPT (paste into a fresh agent in this worktree)

> You're picking up PR #6405 (`manaflow-ai/cmux`) on branch `feat-ios-paired-mac-backup`, in the
> worktree `/Users/lawrence/fun/cmuxterm-hq/worktrees/feat-ios-paired-mac-backup`. Read
> `AUTOREVIEW-HANDOFF.md` at the worktree root first — it has the full context and the 63 open
> Aziz-policy findings.
>
> State: the structured Codex correctness review is CLEAN; ~63 `cmux-policy-check` P2 findings remain
> (39 documentation, 14 file-org, 7 package-design, 2 testability, 1 concurrency). The PR is large
> (backup + multi-Mac + customizations + teams + production presence + build-channel pills).
>
> Do this:
> 1. Re-run `/Users/lawrence/fun/cmuxterm-hq/skills/autoreview/scripts/cmux-policy-check --mode branch
>    --base origin/main` to get the current list.
> 2. Work the findings per the "Triage guidance" in AUTOREVIEW-HANDOFF.md: FIX all documentation
>    findings (add `///` docs); FIX easy file-org splits; REJECT-with-evidence the namespace-enum
>    conversions (pure stateless function namespaces) and the `DeviceTreeView` poll-timer `Task.sleep`
>    (bounded, cancelled-on-dismiss UI refresh the user requested) with brief inline comments.
> 3. Compile-check the iOS packages locally WITHOUT the slow cloud build:
>    `SDK=$(xcrun --sdk iphonesimulator --show-sdk-path); cd Packages/iOS/CmuxMobileShellUI &&
>    swift build --sdk "$SDK" --triple arm64-apple-ios18.0-simulator` (GhosttyKit is already built in
>    this worktree). Run `swift test` on the touched packages.
> 4. Re-run autoreview (`skills/autoreview/scripts/autoreview --mode branch --base origin/main`) until
>    the helper exits 0 or every remaining finding is a documented conscious exception.
> 5. Do NOT auto-merge: this is an iOS runtime PR and needs Lawrence's explicit dogfood approval
>    ("looks good to merge"). Also remove this AUTOREVIEW-HANDOFF.md before merge.
> 6. Optional but recommended: propose splitting teams / presence-prod / build-channel into follow-up
>    PRs.
>
> Build commands (cloud, offload local CPU): mac `./scripts/reload-cloud.sh --tag teams`; iOS
> `./ios/scripts/reload-cloud.sh --tag teams --device-id E4058DA9-F4C7-52DD-951D-0354061B8E89`.
> Worker dev deploy (safe): `cd workers/presence && bunx wrangler deploy --config wrangler.dev.toml`
> (NEVER `--name`, it steals the prod domain).
