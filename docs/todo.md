# To do

Tracks what's done and what's pending. Details and reasons: `backup-plan.md` (phases, decisions D1–D8),
exact commands: `runbook.md`. Mark items `[x]` with the date; add a short note when something changes.

Last updated: 2026-10-14

## Done
- [x] Steam library in its own subvolume (runbook step 1), 2026-10-14
- [x] Snapper, snap-pac, grub-btrfs installed and checked (runbook step 2a–2e), 2026-10-14
- [x] Repo on GitHub (`hatzibod/earth-tools`), first commit pushed, 2026-10-14
- [x] Backup plan written (`backup-plan.md`), 2026-10-14

## Decisions
- [x] D1 rest-server or SFTP: **rest-server, append-only**, decided 2026-10-14
- [x] D2 back up twice, `restic copy`, or mirror: **back up twice**, decided 2026-10-14
- [x] D3 where prune runs: **DSM scheduled task on each NAS**, decided 2026-10-14
- [x] D4 TLS or Tailscale encryption: **tailnet addresses, TLS later**, decided 2026-10-14
- [x] D5 retention: **14 daily, 8 weekly, 12 monthly via `--keep-within-*`**, decided 2026-10-14
- [x] D6 one repo password or one each: **one each, in Bitwarden**, decided 2026-10-14
- [x] D7 Backrest: **later**, decided 2026-10-14
- [x] D8 `snbk` to a USB disk: **later**, decided 2026-10-14

## Phase 0: finish what's open (earth)
- [ ] Snapper reboot test: boot snapshot #4 from GRUB, `cowsay hi` works, reboot normally
- [ ] Mark runbook step 2 done

## Phase 1: gather facts (read-only)
- [ ] earth: size of the backup set (`/home` minus Steam games and `.cache`, `compatdata/`, `userdata/`, `/etc`)
- [ ] alexandria: DSM version, free space, Container Manager, tailnet name
- [ ] Pharos: model, DSM version, free space, filesystem, Container Manager support, existing jobs
- [ ] earth upload / Pharos download speed → time for the first Pharos backup
- [ ] Tailscale ACL: can earth reach both NASes on the rest-server port

## Phase 2: prepare restic (earth)
- [ ] Install restic
- [ ] Generate repo passwords, store in Bitwarden and `/root/.config/restic/`
- [ ] Offline escrow copies (fire safe, second location)

## Phase 3: rest-server on alexandria
- [ ] Shared folder `restic`
- [ ] rest-server container (pinned version, `--append-only --private-repos`)
- [ ] User `earth`
- [ ] DSM firewall: port 8000 from earth and the tailnet only
- [ ] Check from earth with `curl`

## Phase 4: first backup to alexandria (earth)
- [ ] Include and exclude files in `config/`
- [ ] Dry run: saves in, game installs and `.cache` out
- [ ] `restic init`, first backup by hand
- [ ] `restic check` and restore test (one file, one Proton save folder)

## Phase 5: pruning and checks on alexandria
- [ ] Weekly scheduled task: `forget --prune`, `check --read-data-subset=5%`
- [ ] E-mail on failure
- [ ] Run once by hand, snapshots still listed

## Phase 6: automate earth → alexandria (earth)
- [ ] Backup script and `pacman -Qqe` dump
- [ ] `bats` tests (needs OK for `bats`)
- [ ] systemd service and timer (daily, `Persistent=true`)
- [ ] Failure notification
- [ ] Check: a new snapshot appears the next day

## Phase 7: rest-server on Pharos
- [ ] Tailscale ACL: earth → Pharos:8000 (homelab change)
- [ ] Shared folder, container, user, firewall (as phase 3)
- [ ] Check from earth with `curl`

## Phase 8: first backup to Pharos (earth)
- [ ] `restic init` with its own password
- [ ] First backup by hand
- [ ] `restic check` and restore test
- [ ] Add to the script and timer as a separate run

## Phase 9: pruning and checks on Pharos
- [ ] Scheduled task, e-mail on failure, run once by hand (as phase 5)

## Phase 10: routine
- [ ] Monthly: logs and `restic snapshots` on both repos (first one due: one month after phase 8)
- [ ] Quarterly: restore test from each repo
- [ ] Yearly: Bitwarden and offline passwords still open both repos

## Phase 11 (optional): `snbk` to a btrfs USB disk
- [ ] Depends on D8

## Loose ends
- [ ] Commit the desktop-table row in `ai-tooling/claude-config/projects/desktop/CLAUDE.md`
- [ ] Link this project from the homelab "backup strategy for every machine" item
- [ ] Elite on Proton Experimental vs `machine.md` saying Proton 11.0: which is right
- [ ] Folder `earth-backup` vs remote `earth-tools`: rename or keep
- [ ] Making a booted snapshot permanent: untested
