# Runbook

Steps Nikos runs or OKs. Each step ends with a check.

## 1. Move the Steam library into its own subvolume

**Status 2026-10-14:** done. Copy and swap took 8.6 s (740,387 entries, file lists match, disk use
unchanged). Steam and the Elite launcher (Proton, signed in through Steam) start fine from the new
subvolume. `Steam.old` deleted.

Why: `@home` snapshots then skip the ~895 GB of games, which can be downloaded again. The path stays
`~/.local/share/Steam`; nothing moves for Steam. No root needed: it's a normal user's subvolume in
their own home.

Preflight (checked 2026-10-14): Steam closed, `Steam` is a plain directory in `@home` (inode ≠ 256), no
subvolumes or mounts nested inside, 2.3 TB free.

```bash
set -euo pipefail
cd ~/.local/share
pgrep -x steam && { echo "Steam is running"; exit 1; }
btrfs subvolume create Steam.new
cp -a --reflink=always Steam/. Steam.new/        # shares data blocks: ~no extra space
diff <(cd Steam && find . | sort) <(cd Steam.new && find . | sort) && echo "file lists match"
mv Steam Steam.old
mv Steam.new Steam
```

Check:
- `stat -c %i ~/.local/share/Steam` prints `256` (it's a subvolume root).
- Start Steam and launch two games, one of them via Proton (e.g. Elite Dangerous): settings, bindings and
  saves are there.

Then, and only then: `rm -rf ~/.local/share/Steam.old`.

Rollback (before deleting `Steam.old`): `mv Steam Steam.broken && mv Steam.old Steam`.

Note: game saves in `steamapps/compatdata/` and `userdata/` are no longer in `/home` snapshots. Restic
must back them up (step to come).

## 2. Snapper: snapshots before and after every pacman run, bootable from GRUB

**Status 2026-10-14:** 2a–2e done. Checked: both configs, `/.snapshots` on `@snapshots`, quotas off,
timers on (boot snapshot #1 at 10:58), `PRUNENAMES` includes `.snapshots`, the EOS overlay hook is in the
initramfs, and the cowsay install and removal each made a pre/post pair (#2–5), which `grub-btrfsd`
added to `grub-btrfs.cfg` on its own. **Still open: the reboot test** (submenu shows, a snapshot boots).

Run as root (`sudo -i`). Sources: [ArchWiki – Snapper](https://wiki.archlinux.org/title/Snapper),
[grub-btrfs README](https://github.com/Antynea/grub-btrfs).

### 2a. Packages

```bash
pacman -S --needed snapper snap-pac grub-btrfs inotify-tools btrfs-assistant
```

`snap-pac` starts working right away, but it does nothing until the `root` config exists (2c).
`btrfs-assistant` is an optional GUI for browsing and restoring snapshots.

### 2b. `@snapshots` subvolume at `/.snapshots`

This is the wiki's suggested layout. Snapshots live outside `@`, so `@` can be replaced by a snapshot from
a live USB without losing the other snapshots.

```bash
mkdir -p /mnt/btrfs-top
mount -o subvolid=5 /dev/disk/by-uuid/7ec9f8b5-c6b8-4dba-ac70-59d38cb53054 /mnt/btrfs-top
btrfs subvolume create /mnt/btrfs-top/@snapshots
umount /mnt/btrfs-top
```

### 2c. Snapper configs

```bash
snapper -c root create-config /          # also creates a nested /.snapshots subvolume we don't want
btrfs subvolume delete /.snapshots
mkdir /.snapshots
echo 'UUID=7ec9f8b5-c6b8-4dba-ac70-59d38cb53054 /.snapshots    btrfs   subvol=/@snapshots,noatime,compress=zstd 0 0' | tee -a /etc/fstab
systemctl daemon-reload
mount /.snapshots
chmod 750 /.snapshots

# root: no hourly timeline; pacman pairs and boot snapshots, keep about 10
snapper -c root set-config TIMELINE_CREATE=no NUMBER_LIMIT=10 NUMBER_LIMIT_IMPORTANT=10 ALLOW_USERS=nikos

# home: hourly for a day, daily for a week (Steam is a nested subvolume, so it's skipped)
snapper -c home create-config /home
snapper -c home set-config TIMELINE_CREATE=yes TIMELINE_LIMIT_HOURLY=24 TIMELINE_LIMIT_DAILY=7 \
    TIMELINE_LIMIT_WEEKLY=0 TIMELINE_LIMIT_MONTHLY=0 TIMELINE_LIMIT_YEARLY=0 ALLOW_USERS=nikos

systemctl enable --now snapper-timeline.timer snapper-cleanup.timer snapper-boot.timer
```

Btrfs quotas stay off: the wiki links them to serious slowdowns. Check with `btrfs qgroup show /`,
which should say quotas are not enabled.

### 2d. Keep plocate out of the snapshots

In `/etc/updatedb.conf`, change `PRUNENAMES = ".git .hg .svn"` to:

```
PRUNENAMES = ".git .hg .svn .snapshots"
```

### 2e. Snapshots in the GRUB menu

```bash
systemctl enable --now grub-btrfsd
grub-mkconfig -o /boot/grub/grub.cfg
```

Booting a read-only snapshot needs a writable overlay. EndeavourOS's `eos-dracut` package already ships one:
the dracut module `91btrfs-snapshot-overlay` puts a RAM overlay on top whenever the root is a read-only
Btrfs snapshot. That makes grub-btrfs's own overlay setup (`rd.live.overlay.*`) unnecessary. Check that the
module is in the initramfs:

```bash
lsinitrd /boot/initramfs-linux.img | grep snapshot-overlay
```

If nothing is printed, add it and rebuild:

```bash
echo 'add_dracutmodules+=" btrfs-snapshot-overlay "' > /etc/dracut.conf.d/btrfs-snapshot-overlay.conf
dracut-rebuild
```

### Checks
- `snapper list-configs` shows `root` → `/` and `home` → `/home`.
- `findmnt /.snapshots` shows `[/@snapshots]`.
- Install a small package (`pacman -S cowsay`), then `snapper -c root list`: a `pre` and a `post` snapshot
  with the pacman command as description. Remove it again (`pacman -Rns cowsay`): another pair.
- `grub-mkconfig` output lists the snapshots it found. After a reboot the GRUB menu has an
  "Arch Linux snapshots" submenu.
- Boot one snapshot from that submenu: the desktop comes up (changes vanish on reboot). Reboot normally.
- An hour later, `snapper -c home list` shows a `timeline` snapshot.

### Recovering from a bad update
1. Boot the `pre` snapshot from the GRUB submenu.
2. If it works and you want to keep it: restore `@` from it with btrfs-assistant, or by hand from a live
   USB as in the wiki ("Restoring / to its previous snapshot"). Not yet tested here.
