# Open decisions

## Restic target on the Synology: rest-server vs SFTP (open, 2026-10-14)

**Proposed: rest-server in append-only mode.** Not decided yet.

For rest-server:
- **Append-only** (`--append-only`): earth can add backups but not delete or change them. The README says it
  "prevents an attacker from wiping your server backups when access is gained to the server being backed up."
  With SFTP, earth's SSH key can delete the whole repo, and so can ransomware or a mistaken command.
- **Faster:** the README says REST beats SFTP, because SFTP moves data in chunks of at most 32 KiB, each
  one acknowledged.
- **No dropped connections:** the restic docs warn that SFTP connections drop while restic scans lots of
  unchanged data. The fix is `ServerAliveInterval 60` / `ServerAliveCountMax 240`.

Against it:
- One more container on the Synology. Docker already runs there (`homelab/research/01-current-setup.md`).
- In append-only mode earth can't prune. `forget --prune` has to run somewhere with full access, such as a
  Synology scheduled task or a separate admin user.
- Needs TLS, or the basic-auth password crosses the network in clear text. Restic encrypts the data itself.
  Possible certificate source: the `certmint` pipeline (`homelab/synology/runbook-lets-encrypt-acme-sh.md`).

For SFTP: needs no new service; DSM has it built in.

Not yet checked: how the rest-server Docker image enables append-only, and the DSM setup for it.

Sources: [rest-server README](https://github.com/restic/rest-server),
[restic docs: preparing a repository](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html).

## Backrest web UI on top of restic (TBD, 2026-10-14)

Plain systemd timers work either way, and Backrest can be added later on the same repo. The homelab decision
log already lists "Restic/Backrest" as the backup recommendation (`homelab/decisions.md`). Upstream's latest
release is v1.14.1, dated 2026-07-12 ([newreleases](https://newreleases.io/project/github/garethgeorge/backrest/release/v1.14.1)).
Not checked: whether Backrest is in the AUR.
