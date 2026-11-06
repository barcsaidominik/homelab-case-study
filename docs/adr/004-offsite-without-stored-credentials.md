# ADR-004: Offsite copy: manual weekly upload, no cloud credentials stored on the host

- **Status:** accepted (replaces an earlier "no offsite copy, accepted risk" decision)
- **Date:** 2026-10-29
- **Decided by:** me; the export job and the read-only share were proposed by the assistant

## Context

The nightly backup ([ADR-001](001-nightly-backup-without-downtime.md)) lands on a USB disk attached to the server,
the same disk that holds the media library. Theft, fire, a power surge, or ransomware with admin rights would take the
server and its backup together. At first I accepted that risk and postponed the offsite copy; a week later I
reopened the question.

The repository is small (10 GB at the time, growing by a few hundred MB a day, expected to settle at 15–30 GB) and
encrypted with a password the server does not need to share with anyone. The question was not where to store it, but
**who is allowed to write to the offsite copy**.

## Options

1. **A paid object store (B2, Hetzner Storage Box) with an automatic `restic copy` from the server.** Fully
   automatic, but the server then holds credentials that can write to, and usually delete, the offsite copy. Ransomware
   on the server gets them too.
2. **A free consumer cloud drive with a sync client or rclone on the server.** Same problem, plus a sync client that
   happily mirrors encrypted or deleted files.
3. **An offline disk at a relative's home, swapped monthly.** Strong protection, but it depends on a habit that two
   households have to keep.
4. **A manual upload from my desktop, through the browser.** The server has no cloud credentials at all; the desktop
   can only *read* the backup.

## Decision

Option 4.

- The server exposes the backup folder through a **read-only share**, using a dedicated non-admin account (share and
  NTFS permissions both read-only). The desktop can copy the repository but cannot change or delete it.
- A weekly scheduled task on the desktop packs the repository into a single archive (no compression: the data is
  already encrypted and deduplicated), verifies the file count, keeps the last two archives, and shows a notification.
  It skips the server's backup window and does not run while restic holds a lock.
- I upload the archive through the provider's web interface: weekly, and right away after a big change.

No sync client, no stored token, no automation that can delete the offsite copy.

## Verification

- The archive's file count is checked against the repository before it is offered for upload.
- The repository password is in my password manager, so the offsite copy can be restored without the server and
  without the desktop. A quarterly check is planned: download the latest upload and run `restic check` on it.

## Consequences

- The offsite copy is as old as my last upload; if I forget, the recovery point outside the house grows. The weekly
  notification is the reminder, and I accept that it is a manual step.
- No new paid service.
- An attacker who controls the server cannot reach the offsite copy, and one who controls the desktop can read an
  encrypted repository but not the password.
