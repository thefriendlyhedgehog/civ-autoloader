# CFS stops at `FSP root: not verified`

Recorded 2026-11-15 on the reference console (Switch 2, fw 23.0.1, Civ 7 1.0.0
cartridge, CivSwitch Remote JS Loader on `:9026`).

Status: **open**. CivSwitchFS completes the cache alias and fails the FSP
identity stage on every attempt, so the kaiHEN stage never starts. The s8n
`civ7-fs-kaihen` engine performs the same promotion on the same console and
succeeds, so this is a property of the CFS inspection path, not of the console.

## Symptom

```
Address qualification: read verified; … ; alias recheck=… ; period=passed
Metadata: link checks=… ; valid pages=… ; …
FSP root: not verified; scans=…; PID not read; more candidates may qualify=…
Stopped after 10 attempts; user storage access was not verified. Qualified windows: 59/70 (84%).
Restart the game before another run.
```

Attempts that flip the canary take ~30 s and end here; attempts that miss take
~4 s. With a clean save, 5 of 10 attempts flipped and all 5 failed this stage.

## What the stage is

1. `discovery.mjs` qualifies a metadata page, then calls
   `inspectFspIdentityTargetsReadOnly` (`src/core/helpers/scan.mjs:23`).
2. That returns `identityResult.ok`, assigned to `rootStageVerified`
   (`src/core/steps/discovery.mjs:604-645`).
3. `runPromotionStep` only promotes when
   `addressReadVerified && rootStageVerified && verifiedIdentityTargets && …`
   (`src/core/promotion.mjs:39`).
4. Without it, no PID is set to 1, the user-storage probe never runs, and the
   autoloader's kaiHEN stage is not reached.

The reason string is first-class: `root-identity-not-verified`
(`src/core/log-format.mjs:626`), and the stage exposes
`rootCandidateScans`, `rootPidRead`, `rootChunksRead` and
`rootCandidateRetryable` for diagnosis.

## Evidence from the console

| time | save | attempts | canary flips | outcome |
|---|---|---|---|---|
| 12:33 | stock CivSwitch trigger | 10 | 0 | windows 122/135 (90 %), canary never flipped |
| 12:40 | stock trigger after save restore | 10 | 0 | windows 80/87 (92 %), canary never flipped |
| 13:10 | recreated (CivSwitchLDN install) | 10 | 5 | every flip failed `FSP root: not verified` |

The recreate came from `ItsNerisma/CivSwitchLDN` (`npm start`, payload over LDN,
save template over UDP 9027, `INSTALL DONE`), which replaced the single-player
saves. That fixed the canary half — the alias is installed and the metadata page
qualifies — and left the identity half failing.

The same console then ran the `s8n/civ7-fs-kaihen` engine to a verified 4/4 and
a live HEN (`switch2-WraBwn`, acquisition 135 s) on the recreated save, so
`fsp-srv` promotion is possible here with a different implementation.

## Not the cause

- **Save geometry.** A recreated save changes the canary result (0 → 5 flips)
  but not this stage.
- **Timing.** 84–92 % qualified windows across three runs; retuning does not
  move it.
- **Corrupted state.** Every attempt cleans up (`Cleanup complete`) and the
  next attempt runs from scratch.

## Next steps

1. Instrument `inspectFspIdentityTargetsReadOnly`: log each candidate, the
   chunks read, and why `rootPidRead` stayed false. The counters exist but are
   only summarised.
2. Compare the identity-target layout it expects against what the s8n engine
   matches (`engine/switch2/ro/*`, donor `main.npdm`, `smServiceOffset`,
   `sessionManager` fields in `targets/civ7/target.json`).
3. With a held RPC session, dump `fsp-srv`'s domain table and the candidate
   objects the scan considers, and diff them against the check in
   `scan.mjs:unusedCacheTail` / `rememberRetainedOwner`.

## Related

- `docs/compatibility.md` — the tested firmware/title combination.
- `packages/orchestrator/README.md` — where this stage sits in the pipeline.
- The keep-loader build variant (`CFS_KEEP_LOADER=1` in
  `third_party/CivSwitchFS/tools/persist-build.mjs`) keeps `:9026` alive after
  the payload, so a console with a persistent CFS save can still be driven and
  reinstalled over the network when this stage fails.
