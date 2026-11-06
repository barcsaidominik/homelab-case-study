# ADR-001: Nightly restic backup with VSS and online database backups, without stopping services

- **Status:** accepted
- **Date:** 2026-10-21
- **Decided by:** I set the requirements (daily, no downtime); the tool choice was proposed by the assistant and approved by me

## Context

Before this decision the server had no systematic backup. It runs about twenty services, most of them with an SQLite
database: some natively on Windows, some in Docker Desktop containers with their data on bind mounts or volumes.
The family uses the media server in the evening and the server updates itself at night, so there is no maintenance
window in which every service can be stopped. I wanted a recovery point of at most 24 hours for everything that
matters, and I wanted the backup to be encrypted, deduplicated and testable, because the backup disk is a USB drive
that could be lost or stolen.

## Options

1. **Stop the services, copy the files, start them again.** Simple and consistent, but it means a nightly outage
   and a failure mode where a service does not come back.
2. **Copy files while the services run.** No outage, but a copied SQLite file can be torn mid-write; the backup
   looks fine until the day you need it.
3. **restic with a Volume Shadow Copy snapshot, plus each database exported with the SQLite online backup API.**
   VSS gives a point-in-time view of the files; the backup API gives a consistent copy of a database that is being
   written to.
4. **Windows Server Backup / a commercial tool.** Image-level backups are heavy for a laptop and do not deduplicate
   across days the way restic does.

## Decision

Option 3. A scheduled task runs every night at 03:30 as SYSTEM:

1. For every database, an online backup into a staging folder: with Python's `sqlite3` backup API for the native
   apps, and *inside* the container for the containerised ones, because SQLite locking over a Windows ↔ Linux bind
   mount is not reliable. Each copy is checked with `PRAGMA integrity_check`, and the result is written to a manifest.
2. Read-only exports of the configuration that is not in files: scheduled tasks, firewall rules, SMB shares, the
   list of installed programs, container metadata and image digests.
3. `restic backup` of the data folders and the staging folder with a VSS snapshot. Caches, logs and anything that is
   rebuilt automatically (thumbnails, trickplay images) are excluded.
4. Retention: `restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6`, `prune` weekly.

The restic password exists twice: in a file on the server that only SYSTEM and Administrators can read, and in my
password manager, so the repository can be restored without the server. The script refuses to run if the backup
disk or the repository is missing; it never creates a target on another drive.

## Verification

- The first run and every run since report to the central log (duration, new and changed files, size). A typical
  night takes under three minutes for about 15 GB of data.
- A health check that runs every six hours raises an incident if there was no successful backup in the last 26 hours.
- The restores are tested, not assumed: see [ADR-003](003-restore-testing.md).

## Consequences

- No nightly outage, and no service ever has to be stopped for a backup.
- Every new service needs a decision about *how* its data is backed up (online API, app-native export, or VSS copy
  with a recorded risk). This is part of adding a service, not an afterthought.
- The backup and the media library live on the same physical disk. A failure of that disk takes both, so the offsite
  copy ([ADR-004](004-offsite-without-stored-credentials.md)) is the real protection against it.
