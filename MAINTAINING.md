# Maintaining this fork — pin & patch on breakage

This repo is a fork of
[Gielz1986/Zendure-HA-zenSDK](https://github.com/Gielz1986/Zendure-HA-zenSDK)
and the deploy source for the two-battery setup. The integration is local,
cloud-free and feature-complete for this install, so there is **no routine
upstream tracking**. The policy:

1. **Stay pinned.** `main` = known-good upstream `v20260824` + the shim + the
   fleet layer. While it works, leave it alone. Do not merge upstream on a
   schedule or chase features.
2. **Keep upstream reachable as cheap insurance.** The `upstream` remote stays
   configured so that when something breaks you can
   `git fetch upstream --tags`, diff against the latest release, and
   cherry-pick the specific fix. That is its only job.
3. **React to trip-wires, not a calendar.** Act only when:
   - `binary_sensor.zendure_fleet_vendor_contract_ok` goes **off** (unit 2 is
     then held idle by the interlock; its `missing` attribute names the broken
     entity), or
   - after an HA core upgrade, the HA logs show deprecation/breakage warnings
     touching templates, `rest`/`rest_command`, or Jinja. **Main watch item:**
     read the "Backward-incompatible changes" section of the HA release notes
     after each core update.

## Repo layout (unchanged rules)

| Branch | Purpose |
|---|---|
| `main` | Deploy branch: pinned upstream + shim + fleet layer. HA deploys from here only. |
| `pr/fix-nordpool-isoformat` | Throwaway, upstream PR for the nordpool string fix (EN+NL). Never deploy. |
| `pr/multi-device-fleet` | Throwaway, best-effort upstream PR for the fleet layer. Never deploy. |

Remotes: `origin` = your fork (`https://github.com/<you>/Zendure-HA-zenSDK.git`),
`upstream` = Gielz1986 (read-only insurance).

Two standing rules that make patching cheap:

- **Additions live in new files only** (`packages/zendure2.yaml`,
  `packages/zendure_fleet.yaml`, `FLEET.md`, `MAINTAINING.md`, `patches/`).
  Never move fleet code into vendor files.
- **The shim stays isolated**: the only vendor-file modification is the
  nordpool string fix, one clearly-messaged commit, exported to
  `patches/shim-nordpool-string.patch` (`git apply` fallback). The patch
  covers the EN package (what this install deploys); the upstream PR branch
  fixes EN+NL.

## The two real break sources

### A. Home Assistant core breaking change

Symptoms: template errors, dead `rest` sensors, automation warnings after an
HA upgrade.

1. Check whether upstream already adapted:
   ```sh
   git fetch upstream --tags
   git log upstream/main --oneline            # look for a fix release
   git diff v20260824..<new-tag> -- "Global (EN) Integration/"
   ```
2. Pull just the fix — cherry-pick the commit, or take the changed vendor file
   and re-apply the shim:
   ```sh
   git cherry-pick <fix-sha>
   # or:
   git checkout <new-tag> -- "Global (EN) Integration/packages/zendure_gielz1986_global.yaml"
   git apply patches/shim-nordpool-string.patch
   git add "Global (EN) Integration/packages/zendure_gielz1986_global.yaml" && git commit
   ```
3. **Also check the fleet layer**: `packages/zendure2.yaml` and
   `packages/zendure_fleet.yaml` use the same template/automation/REST surface
   as the vendor package, so the same deprecated API likely appears there.
   Fix it in place (new files = no conflict concerns).

### B. Zendure firmware change

Renamed/removed property, added auth, or a locked-down local API (cf. the
401-on-grid-charge issue on the 800 Pro2). Symptoms: REST sensors
unavailable or wrong, writes rejected, contract sensor off.

1. Check upstream (and its issue tracker) for an adaptation and pull it as in
   A.2 — firmware quirks almost always hit the whole user base, so upstream is
   the fastest source of a fix.
2. Mirror any property rename into the fleet layer's REST sensors/commands and
   the contract list at the top of `zendure_fleet.yaml` (the designed
   maintenance point).

## After any patch, before deploying

1. HA config check (Developer Tools → YAML → Check Configuration).
2. `binary_sensor.zendure_fleet_vendor_contract_ok` is **on**.
3. `sensor.dynamic_nordpool` is not `unavailable` and `raw_today` is populated
   (confirms the shim survived).
4. Deploy as usual (copy the three package files into `config/packages/`,
   restart). Other local packages (`zendure_helpers.yaml`, morning sweep,
   strategy-D shadow) live outside this repo and are untouched.

## Accepted trade-off

Deferring merges means a forced merge, when it eventually comes, is larger
than a routine one — possibly spanning several releases at once (resolve the
shim hunks, or re-apply the patch; add/add conflicts on our new file paths are
possible but unlikely — rename ours if it happens). For a solo home setup
that occasional catch-up cost is cheaper than a merge-and-verify cycle per
upstream release.

## One-time setup / upstream PRs

Fork + remotes + pushes, and the two `gh pr create` commands for the
nordpool-fix and fleet-layer PRs, are unchanged:

```sh
gh repo fork Gielz1986/Zendure-HA-zenSDK --clone=false
git remote add origin https://github.com/<you>/Zendure-HA-zenSDK.git
git push -u origin main
git push origin pr/fix-nordpool-isoformat pr/multi-device-fleet

gh pr create --repo Gielz1986/Zendure-HA-zenSDK \
  --head <you>:pr/fix-nordpool-isoformat \
  --title "Fix dynamic_nordpool raw_today/raw_tomorrow for string-based period starts" \
  --body "Price sensors that emit ISO strings for period start/end break the raw_today/raw_tomorrow reconstruction of sensor.dynamic_nordpool (strings have no .isoformat()). Wrapping the four sites per package in as_datetime() passes datetimes through unchanged and parses strings, so both source types work. Applied to the EN and NL packages."

gh pr create --repo Gielz1986/Zendure-HA-zenSDK \
  --head <you>:pr/multi-device-fleet \
  --title "Add optional pure-HA multi-device fleet layer (new files only)" \
  --body "Adds an optional second-unit coordinator in new files only (packages/zendure2.yaml, packages/zendure_fleet.yaml, FLEET.md) — no changes to the existing package or automation. Unit 1 stays under the stock 5s loop; unit 2 runs as a slower SoC-weighted baseload setter on the same meter signal. Offered as an alternative to the Node-RED proxy for users who want to stay YAML-only (ref #166). Feel free to treat this as documentation if you prefer the proxy as the official path."
```

Both PRs are best-effort; nothing here depends on them merging. If the
nordpool fix does merge, the local shim resolves away whenever a fix release
is eventually pulled — delete the patch file and note it here.
