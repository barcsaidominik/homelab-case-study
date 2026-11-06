# ADR-007: Per-service non-admin accounts; virtual service account for the monitor

- **Status:** accepted
- **Date:** 2026-10-31 (the monitor: 2026-11-02; the folder permissions: 2026-11-02)
- **Decided by:** me, after a read-only security assessment prepared with the assistant

## Context

After [ADR-005](005-unattended-startup-without-autologon.md) the long-running apps (the media servers, a download
client) started from scheduled tasks under my admin account, with the task's run level set to "limited". A security
assessment found two problems:

1. **"Limited" did not limit anything.** I measured the processes' tokens: with S4U and the limited run level they
   still ran with a full admin token at High integrity. Any remote code execution in a media server, the most
   internet-facing component, would have been admin on the host.
2. **A local privilege escalation path.** The software folder was writable by every authenticated user, and from that
   same folder a SYSTEM service and a SYSTEM scheduled task started their binaries. Anything that could run as an
   ordinary account could replace a binary and become SYSTEM on the next start.

## Options

1. **Keep the admin account, rely on the apps being trustworthy.** No work, no protection.
2. **One shared non-admin account for all services.** Better, but one compromised app could read and change the data
   of all the others.
3. **One non-admin account per service**, with the shared rights on a group and explicit denials where a service must
   not reach another one's data. A **virtual service account** (`NT SERVICE\<name>`) where the app is a real Windows
   service: no password at all.

## Decision

Option 3, one service at a time, each with its own migration script that has `Cutover`, `Probe` and `Rollback` modes.

- Every account: not an administrator, interactive and remote sign-in denied, member of a common services group.
- Write access only where the service has to write: its own profile or data folder, the transcode cache, and the media
  library for the apps from which the family deletes watched items (that is intended, and written down as such).
- Deny entries on the system-wide software folder and on other services' data folders, per account.
- The tasks run with a long random password, because S4U cannot be registered for a non-admin account; the script
  generates it, sets it on the account and the task in one step, and nobody needs to know it.
- The monitor service ([homelab-monitor](https://github.com/barcsaidominik/homelab-monitor)) runs as a virtual service
  account: no password, no stored secret, only read access to the results of the privileged checks.
- The software folder's permissions were tightened: inheritance disabled, the "authenticated users" entry removed,
  explicit rights only for the administrators, SYSTEM and the deploy account. The secret files in it (environment
  files, keys) are readable only by those. The previous ACL was exported first, with a rollback script.

## Verification

- After each cutover the token of the new process was checked: not elevated, running as the new account.
- Hardware transcoding was tested *before* the move (a non-admin process in session 0 using the GPU encoder), because
  losing it would have made the change pointless. It worked, and remote playback through it was tested afterwards.
- A reboot without signing in after each step, checked with the startup test script: 70–71 checks, 0 failures.
- Client-side tests on real devices: playback, live TV, deleting from the apps.

## Consequences

- A compromised media server is a non-admin account that cannot touch the system, the other services or the secrets.
- Every new service needs its account, its group membership and its deny entries; the scripts make that repeatable,
  but it is real work.
- A lesson learned: when an admin resets a service account's password, Windows invalidates that
  account's DPAPI-protected data. One app regenerated its encryption keys without data loss, but password resets are
  now done only through the script and only when needed.
