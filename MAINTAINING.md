# Maintaining this fork

This repo is a **maintained fork** of
[Gielz1986/Zendure-HA-zenSDK](https://github.com/Gielz1986/Zendure-HA-zenSDK).
It is the real distribution path for the two-battery setup: HA deploys from
this fork's `main`, upstream features get pulled in tag by tag, and nothing
here depends on any upstream PR being accepted.

## Repo layout

| Branch | Purpose |
|---|---|
| `main` | **Deploy branch.** Upstream content + our additions. HA deploys from here, nothing else. |
| `pr/fix-nordpool-isoformat` | Throwaway: the nordpool string fix (both EN and NL packages) based on the upstream tag, for the standalone upstream bug-fix PR. Never deploy from it. |
| `pr/multi-device-fleet` | Throwaway: the fleet layer based on the upstream tag, for the best-effort multi-device PR. Never deploy from it. |

| Remote | Points to |
|---|---|
| `origin` | Your fork `https://github.com/<you>/Zendure-HA-zenSDK.git` (deploy source) |
| `upstream` | `https://github.com/Gielz1986/Zendure-HA-zenSDK.git` (read-only, sync source) |

What `main` adds on top of upstream, by design:

1. **New files only** for all multi-device code: `packages/zendure2.yaml`,
   `packages/zendure_fleet.yaml`, `FLEET.md`, `MAINTAINING.md`, `patches/`.
   New files cannot conflict with upstream *edits* — this is the whole point
   of the layer-alongside design. **Never move any of this into the vendor
   automation or vendor package files.** (Residual risk: if upstream ever
   introduces a file at one of these exact paths, the merge reports an
   add/add conflict — resolve by renaming our file, e.g.
   `packages/zendure_fleet.yaml` → `packages/zendure_fleet_local.yaml`, and
   updating the deploy copy step. `packages/` at the repo root does not exist
   upstream today, so this is unlikely but not impossible.)
2. **Exactly one modification to a vendor file** — the nordpool string shim:
   4× `item.start/end.isoformat()` → `as_datetime(item.start/end).isoformat()`
   in the `dynamic_nordpool` `raw_today`/`raw_tomorrow` reconstruction of
   `Global (EN) Integration/packages/zendure_gielz1986_global.yaml`.
   It lives in a single, clearly-messaged commit
   (`Fix dynamic_nordpool raw_today/raw_tomorrow for string-based period starts`)
   and is exported to `patches/shim-nordpool-string.patch` as a fallback.
   This is the only **expected** merge-conflict surface. Scope note: the local
   shim and the patch file cover the **EN package only** (that is what this
   install deploys); the upstream PR branch intentionally fixes **EN and NL**
   because a complete fix is more likely to be accepted. If the PR merges, the
   local shim resolves away on the next tag merge.

## One-time setup (first machine / after re-clone)

```sh
# 1. Create the fork on your GitHub account (once, needs `gh auth login`;
#    or use the Fork button on github.com):
gh repo fork Gielz1986/Zendure-HA-zenSDK --clone=false

# 2. Wire up this working copy:
git remote add origin https://github.com/<you>/Zendure-HA-zenSDK.git
git remote add upstream https://github.com/Gielz1986/Zendure-HA-zenSDK.git  # if missing

# 3. Publish:
git push -u origin main
git push origin pr/fix-nordpool-isoformat pr/multi-device-fleet
```

## Upstream sync workflow (pulling in future vendor features)

Sync against upstream **release tags, not raw main** — the vendor ships
versioned releases (e.g. `v20260824`). One tag at a time:

```sh
git checkout main
git fetch upstream --tags
git tag --merged upstream/main | sort | tail -5   # see what's new
git merge <next-tag>                               # e.g. git merge v20261101
```

Expected conflicts are the 4 shim hunks in
`zendure_gielz1986_global.yaml` (plus, theoretically, an add/add if upstream
introduces one of our new file paths — see above). Resolve the shim hunks by
keeping the
`as_datetime(...)` form, or — if the merge gets messy — take the upstream
version of the file and re-apply the shim:

```sh
git checkout <next-tag> -- "Global (EN) Integration/packages/zendure_gielz1986_global.yaml"
git apply patches/shim-nordpool-string.patch
git add "Global (EN) Integration/packages/zendure_gielz1986_global.yaml"
git commit
```
(Add the one file explicitly — never `git add -A` during a merge, it can
sweep in unrelated workspace changes.)

If upstream has merged the nordpool fix PR, the shim hunks vanish on their
own: the merge resolves to identical content and the local patch is retired —
delete `patches/shim-nordpool-string.patch` and note it here.

### Post-sync checks (before deploying)

1. **HA config check**: Developer Tools → YAML → Check Configuration (or
   `ha core check`) against the synced files.
2. **Vendor contract**: `binary_sensor.zendure_fleet_vendor_contract_ok` must
   be `on`. If `off`, its `missing` attribute names the renamed/removed vendor
   entity — update the contract list and the matching references at the top of
   `packages/zendure_fleet.yaml` to the new name. That list is the designed,
   single maintenance point of the fleet layer.
3. **Shim still effective**: `sensor.dynamic_nordpool` is not `unavailable`
   and its `raw_today` attribute is populated.
4. Deploy via the normal workflow (copy/symlink the three package files into
   `config/packages/`, restart HA). Only `zendure_gielz1986_global.yaml`,
   `zendure2.yaml` and `zendure_fleet.yaml` come from this repo — other local
   packages (`zendure_helpers.yaml`, `zendure_morning_sweep.yaml`,
   `zendure_strategy_d_shadow.yaml`, …) live outside it and are untouched.

## Upstream PRs

Both PR branches are based on the upstream tag so Gielz sees clean, minimal
diffs:

```sh
# Small standalone bug fix — realistic chance of being merged:
gh pr create --repo Gielz1986/Zendure-HA-zenSDK \
  --head <you>:pr/fix-nordpool-isoformat \
  --title "Fix dynamic_nordpool raw_today/raw_tomorrow for string-based period starts" \
  --body "Price sensors that emit ISO strings for period start/end break the raw_today/raw_tomorrow reconstruction of sensor.dynamic_nordpool (strings have no .isoformat()). Wrapping the four sites per package in as_datetime() passes datetimes through unchanged and parses strings, so both source types work. Applied to the EN and NL packages."

# Multi-device fleet layer — best-effort / documentation. Upstream's answer
# to multi-device is the Node-RED proxy (issue #166), so assume this will NOT
# merge. The fork is the distribution path; nothing blocks on this PR.
gh pr create --repo Gielz1986/Zendure-HA-zenSDK \
  --head <you>:pr/multi-device-fleet \
  --title "Add optional pure-HA multi-device fleet layer (new files only)" \
  --body "Adds an optional second-unit coordinator in new files only (packages/zendure2.yaml, packages/zendure_fleet.yaml, FLEET.md) — no changes to the existing package or automation. Unit 1 stays under the stock 5s loop; unit 2 runs as a slower SoC-weighted baseload setter on the same meter signal. Offered as an alternative to the Node-RED proxy for users who want to stay YAML-only (ref #166). Feel free to treat this as documentation if you prefer the proxy as the official path."
```

Refresh a PR branch after a vendor release: rebase it onto the new tag
(`git checkout pr/... && git rebase <new-tag>`), force-push to `origin`.

## Rules of thumb

- `main` is sacred: deployable at every commit.
- Vendor files: never edit beyond the one shim commit. New behavior goes in
  new files under `packages/`.
- One upstream tag per merge; never merge upstream's moving `main`.
- The contract list in `zendure_fleet.yaml` is where vendor renames get fixed
  — nowhere else should reference vendor internals.
