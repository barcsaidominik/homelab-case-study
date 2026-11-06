# ADR-002: Media library: file list and hash manifest instead of a full copy

- **Status:** accepted
- **Date:** 2026-10-21
- **Decided by:** me (the schedule and the incremental hashing were proposed by the assistant)

## Context

The media library is about 2.9 TB and grows every week. Everything the services know about it (watch history,
metadata, user settings) is small and is covered by the nightly backup ([ADR-001](001-nightly-backup-without-downtime.md)).
The files themselves can be obtained again: losing them costs time, not something irreplaceable.

Backing up 2.9 TB would need a second large disk, hours of I/O every week on the same disk the family streams from,
and an offsite copy that would not fit any free or cheap plan.

## Options

1. **Full copy to a second disk.** Protects against a disk failure, doubles the storage cost, and is still on site.
2. **No backup of the media at all.** Cheapest, but after a failure I would not even know *what* was lost.
3. **A manifest: path, size, modification time and SHA-256 of every file**, kept in the regular backup.

## Decision

Option 3. A weekly job (Sunday 04:30, at most three hours, low I/O priority) writes the manifest. Hashing is
incremental: a file is only hashed again when its size or modification time changed, so the first full pass over
2.9 TB was spread over several Sundays and later runs are much shorter. The latest manifest and eight weekly copies are
kept, and the manifest itself is part of the nightly restic backup.

## Verification

- The job reports to the central log like the backup; the health check warns if the manifest is older than its schedule.
- The manifest answers two questions after an incident: *which* files are gone (to re-obtain them), and, by hashing
  the survivors again on demand, whether a file that is still there is still the same file. The weekly run does not
  re-hash unchanged files, so it does not detect silent corruption by itself.

## Consequences

- A disk failure means re-obtaining the media, which can take days. That is an accepted cost, and it is written down
  so nobody expects otherwise.
- The manifest is a list of the library's contents. It is stored only in the encrypted backup, never in plain text
  offsite.
