# Backup plan for earth (restic to both Synologys)

Written 2026-10-14. This is a plan, not a runbook: each phase becomes a runbook step with exact commands
once its decisions are made. Nothing here has been run.

## Machines

| Name | What | Reached from earth |
|---|---|---|
| **earth** | This workstation. Runs `restic backup`. | — |
| **alexandria** | Local Synology DS1821+, on the LAN (`192.168.178.0/24`) and the tailnet. | LAN or tailnet |
| **Pharos** | Remote Synology, off-site. Already holds the Synology Drive copy. | Tailnet `100.x` only (its site LAN is not advertised, `homelab/decisions.md`) |

## Shape of the setup (decided 2026-10-14)

- earth backs up **twice, into two independent restic repos**, one on alexandria and one on Pharos. A
  problem in one repo (corruption, a bad prune, a lost password) doesn't touch the other.
- Each Synology runs **rest-server in append-only mode** in Container Manager. earth can add snapshots but
  can't delete or change them.
- **Pruning runs on each Synology**, not on earth, from a DSM scheduled task. That task is the only thing
  with delete rights.
- What to back up (from `CLAUDE.md`, step 3): `/home` without Steam games or `~/.cache`, plus
  `Steam/steamapps/compatdata` and `Steam/userdata`, `/etc`, and a daily `pacman -Qqe` list.

## Decisions (all decided by Nikos 2026-10-14, as proposed)

| # | Decision | Decided | Blocks |
|---|---|---|---|
| D1 | rest-server or SFTP (`open-decisions.md`) | rest-server, append-only | phases 3, 7 |
| D2 | Back up twice, `restic copy`, or mirror the repo | back up twice | phases 6, 8 |
| D3 | Where prune runs, and so who holds the repo password besides earth | a DSM scheduled task on each NAS (the NAS can then read earth's backups) | phases 5, 9 |
| D4 | TLS for rest-server, or rely on Tailscale's encryption | earth talks to both NASes over their tailnet addresses; no TLS for now, add a `certmint` cert later | phases 3, 7 |
| D5 | Retention | `forget --keep-within-daily 14d --keep-within-weekly 8w --keep-within-monthly 12m` (restic advises `--keep-within*` for append-only repos) | phases 5, 9 |
| D6 | One repo password for both repos, or one each | one each, both in Bitwarden (escrow plan in `homelab/decisions.md`) | phases 2, 8 |
| D7 | Backrest UI | later; systemd timers first | — |
| D8 | Optional extra: `snbk` to a btrfs USB disk for fast whole-system restore | later | phase 11 |

## Order of work

Phases are in order. Within a phase, the machine is named on each step. **Root** means Nikos runs it.

### Phase 0: finish what's open (earth)
1. earth, root: the snapper reboot test (runbook step 2). This doesn't block the backups, but finishing it
   first keeps one thing open at a time.

### Phase 1: gather facts (read-only, all three machines)
1. earth: size of the backup set: `du -sh` on `/home` minus Steam and `.cache`, on `compatdata/` and
   `userdata/`, and on `/etc`. This sizes both repos and the first upload to Pharos.
2. alexandria: DSM version, free space per volume, whether Container Manager is installed, whether it's on
   the tailnet and under which name.
3. Pharos: model, DSM version, free space and filesystem, whether its model supports Container Manager,
   what alexandria → Pharos jobs already exist. (Some of this is already open in
   `homelab/immich-access-backup.md`.)
4. earth's internet upload speed (and Pharos's download speed): first backup size ÷ the slower of the two
   = how long the first Pharos backup takes.
5. Tailscale: whether the ACL lets earth reach alexandria and Pharos on the rest-server port.

**If Pharos can't run Container Manager**, D1 changes for Pharos only: use SFTP there (built into DSM),
and accept that earth's key can delete that repo.

### Phase 2: prepare restic (earth)
1. earth, root: `pacman -S restic`.
2. earth: generate the repo passwords (D6) and put them in Bitwarden. Store them on earth in
   `/root/.config/restic/` (mode 600, root only), one file per repo. Do the other escrow copies too
   (fire safe, second location): without the password, the backups are unreadable.

### Phase 3: rest-server on alexandria
1. alexandria, DSM: create a shared folder `restic` (no recycle bin, no Synology Drive indexing).
2. alexandria, Container Manager: run `restic/rest-server` with the data volume on that folder at `/data`,
   port `8000`, and `OPTIONS="--append-only --private-repos"`. Pin the image to an exact version.
3. alexandria: `docker exec -it rest_server create_user earth` (prompts for the password; this is the
   HTTP login, separate from the repo password).
4. alexandria, DSM firewall: allow port 8000 only from earth and the tailnet.
5. Check from earth: `curl -u earth http://<alexandria-tailnet-name>:8000/earth/config` answers 404 (no
   repo yet) rather than 401 (login wrong) or a timeout.

### Phase 4: first backup to alexandria (earth)
1. earth: write the include and exclude files (proposed in this repo under `config/`). Exclude
   `steamapps/common`, `shadercache`, `downloading` and `temp` rather than all of `Steam`, so
   `compatdata/` and `userdata/` stay in.
2. earth, root: `restic backup --dry-run -vv` with those files. Check the list: game saves are in, game
   installs and `.cache` are out. Do not use `--one-file-system`: Steam is its own subvolume and would be
   skipped silently.
3. earth, root: `restic init` the alexandria repo, then the first real backup, by hand.
4. earth, root: `restic check`, then a restore test: one file with `restore --include`, and one Proton
   save folder restored to `/tmp` and compared with `diff -r`.

### Phase 5: pruning and checks on alexandria
1. alexandria, DSM Task Scheduler (root task): weekly, run the `restic/restic` image against the repo
   folder directly (no rest-server in between): `forget` with the D5 policy and `--prune`, then
   `check --read-data-subset=5%`. The repo password lives in a root-only file on alexandria (D3).
2. alexandria: the task's e-mail notification on failure (DSM sends it when the script exits non-zero).
3. Check: run the task once by hand; `restic snapshots` from earth still lists the first snapshot.

### Phase 6: automate earth → alexandria (earth)
1. earth: a backup script (backup, then exit non-zero on any error) plus a `pacman -Qqe` dump that runs
   before it. Tests with `bats` (proposed, not yet OKed): the dry-run list includes a known save file and
   excludes a known game file.
2. earth, root: a systemd service and timer, daily, with `Persistent=true` so a missed run happens at the
   next boot. Install them from this repo.
3. earth: a notification when the service fails (`OnFailure=` to a desktop or mail notifier; to decide).
4. Check: the timer fires, `restic snapshots` shows a new snapshot the next day.

### Phase 7: rest-server on Pharos
1. Tailscale ACL (homelab change): allow earth → Pharos:8000.
2. Pharos: the same as phase 3, with its own rest-server user and password.
3. Check from earth over the tailnet: the same `curl` as in phase 3.

### Phase 8: first backup to Pharos (earth)
1. earth, root: `restic init` the Pharos repo with its own password.
2. earth, root: the first backup by hand. It uploads the whole set over the internet (phase 1, step 4
   says how long). It can be stopped and restarted: restic skips data that's already uploaded.
3. earth, root: `restic check` and the same restore test as phase 4.
4. earth: add the Pharos repo to the script and timer from phase 6, as a second, separate run. A failure
   of one repo must not skip the other.

### Phase 9: pruning and checks on Pharos
1. Pharos: the same as phase 5.

### Phase 10: routine (all)
- Monthly, earth: read the timer and task logs; `restic snapshots` on both repos shows recent snapshots.
- Quarterly, earth: a restore test from each repo, alternating which files. Note the date in the runbook.
- Yearly: check that the Bitwarden and offline password copies still open both repos.

### Phase 11 (optional): local btrfs copy for fast whole-system restore
1. earth: a btrfs USB disk and `snbk` sending the snapper `root` and `home` snapshots to it (D8).

## Not checked yet
- Whether Pharos's model runs Container Manager.
- Whether restic's exclude of a parent directory also drops explicitly included children. The proposed
  exclude list avoids the question; phase 4's dry run checks the result either way.
- What earth's upload speed means for the first Pharos backup.
- The exact HTTP answers in the phase 3 `curl` check, the `--keep-within-*` flag names against the installed
  restic version, and DSM's "e-mail only on abnormal exit" option: check each when writing the runbook step.

Sources: [rest-server README](https://github.com/restic/rest-server) (Docker image, `OPTIONS`,
`create_user`, `--private-repos`),
[restic docs: removing snapshots](https://restic.readthedocs.io/en/stable/060_forget.html)
(`--keep-within` for append-only repos, a separate client for prune),
[restic docs: backing up](https://restic.readthedocs.io/en/stable/040_backup.html),
[restic docs: working with repositories](https://restic.readthedocs.io/en/stable/045_working_with_repos.html).
