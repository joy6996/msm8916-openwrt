# Upstream release auto-build (ufi003)

`upstream-watch.yml` follows upstream releases of [akbar-npj/msm8916-openwrt](https://github.com/akbar-npj/msm8916-openwrt) and produces ready-to-flash ufi003 images. It does **not** flash any device, touch EDL, or update the gh-pages apk feed.

## What it does

1. Daily at 01:00 UTC (09:00 Beijing) — or on manual `workflow_dispatch`.
2. Picks the latest upstream `v*` tag (`sort -V`, `test-*` and everything not starting with `v` is ignored).
3. If it differs from `.github/upstream-built` (the dedup/state file), merges the tag into `main`.
   - Merge conflict → job fails, `main` is NOT pushed. Resolve manually.
   - After merge it verifies `msm89xx/config-6.12` still contains
     `CONFIG_IP_ADVANCED_ROUTER=y` and `CONFIG_IP_MULTIPLE_TABLES=y`; missing → fail, no push.
4. Builds **only `ufi003`** by calling `build.yml` via `workflow_call` (`device: ufi003`, `ref: <merged sha>`, `secrets: inherit`).
5. Verifies the artifact before publishing:
   - all 5 tproxy kmods present in the image manifest (`kmod-inet-diag`, `kmod-nf-conntrack-netlink`, `kmod-nf-socket`, `kmod-nft-socket`, `kmod-nft-tproxy` — baked in via `diffconfigs/ufi003` `=y`);
   - `*-squashfs-sysupgrade.bin` larger than 30 MB (guards against truncated artifacts);
   - generates `SHA256SUMS`.
6. Publishes a **pre-release** tagged `auto-ufi003-<upstream tag>` (title says "unverified") with sysupgrade/boot/system/manifest/flash.sh/firmware.zip + SHA256SUMS.
7. Only then writes the tag into `.github/upstream-built` and pushes it. A failed build leaves the state file untouched, so the next run retries.

## Manual trigger / force

```sh
gh workflow run upstream-watch.yml --ref main            # normal: skip if already built
gh workflow run upstream-watch.yml --ref main -f force=true   # rebuild the current tag
```

A forced rebuild of a tag that is already recorded skips the `upstream-built` commit (idempotent) but still publishes a release — `gh release delete` + re-run if you want a clean single release.

## Failure diagnosis

- `gh run view <id> --repo joy6996/msm8916-openwrt --log-failed`
- Merge conflicts are reported by the `check` job; missing kmods / undersized image by the `publish` job.
- GitHub email notifications fire on failure.

## Rollback

- Remove the workflow: delete `.github/workflows/upstream-watch.yml` (and optionally `.github/upstream-built`).
- Revert a bad auto-merge: `git revert -m 1 <merge commit>` on `main`, or reset to the previous commit before the bot push.
- A bad release: `gh release delete auto-ufi003-<tag> --cleanup-tag`.

## Notes / caveats

- **Schedule auto-pause**: GitHub disables `schedule` triggers after 60 days with no repo commits. Push anything to `main` (or re-run the workflow manually) to re-enable it.
- Releases are always **pre-releases** (`--latest=false`) — they never shadow real releases.
- The images are built from an unattended merge; flash them only after a quick look at the upstream changelog. Always keep a known-good backup: `sysupgrade -b /tmp/backup.tar.gz`.
- kmods are baked into the image, so a fresh flash keeps tproxy working without the gh-pages feed.
