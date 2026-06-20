# badusb_ph — fork structure & upstream baseline

This repo is a **standalone fork** of RogueMaster firmware's BadUSB app, adding
`[placeholder]` substitution and per-payload configs. For what the app does, the build
process, and the full changelog, see **[`README.md`](README.md)** (on the `RM` branch).

## Why this isn't a normal GitHub fork

RogueMaster's BadUSB has **no standalone upstream repo** — it lives only inside the
`flipperzero-firmware-wPlugins` firmware tree at `applications/main/bad_usb/`. So there's
no repo to click "Fork" on and no `upstream` git remote. Instead the original is **vendored**:

- **`master`** = a verbatim copy of RM's `applications/main/bad_usb/` at a pinned commit.
  **Never commit here.** (The firmware-shipped `resources/` asset tree — keyboard layouts,
  demo payloads — is omitted; it's deployed by the firmware, not by this FAP.)
- **`RM`** = the original baseline + the badusb_ph changes. This is the default branch.

**Current baseline:** RogueMaster `flipperzero-firmware-wPlugins` commit **`be48837`**
(`RM0620-0008-0.420.0`, API 87.x).

```sh
git diff master..RM            # exactly the badusb_ph changes vs stock RM BadUSB
git diff --stat master..RM
```

## Updating the baseline from a newer RogueMaster release

Because there's no git upstream remote, "pull updates" means **re-vendoring** the folder
from the firmware repo onto `master`, then replaying `RM` on top:

```sh
# 1. Grab just bad_usb from the RM firmware at the new tag/commit (sparse, no full clone)
git clone --depth 1 --filter=blob:none --sparse \
    https://github.com/RogueMaster/flipperzero-firmware-wPlugins.git /tmp/rm-fw
cd /tmp/rm-fw && git sparse-checkout set applications/main/bad_usb && NEW=$(git rev-parse --short HEAD)

# 2. Refresh master from it (drop the firmware-only resources/ tree)
cd /path/to/this/repo
git checkout master
rsync -a --delete --exclude='.git/' --exclude='resources/' \
    /tmp/rm-fw/applications/main/bad_usb/ ./
git commit -am "Re-vendor RogueMaster bad_usb @ $NEW"
git push origin master

# 3. Replay your changes on the new baseline
git checkout RM
git rebase master            # resolve conflicts (RM-API changes land here)
git push --force-with-lease origin RM
```

Update the "Current baseline" commit above when you do this. Use `git merge master`
instead of `rebase` if you'd rather keep a merge commit and not force-push.

**Never commit on `master`** — keeping it a verbatim RM snapshot is what makes
`git diff master..RM` an exact picture of the fork.
