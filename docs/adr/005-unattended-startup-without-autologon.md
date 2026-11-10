# ADR-005: Unattended startup with scheduled tasks and a storage-readiness gate; AutoLogon rejected

- **Status:** accepted
- **Date:** 2026-10-21
- **Decided by:** me (AutoLogon rejected); the task-based design was proposed by the assistant

## Context

After a reboot only real Windows services came back. Everything else lived in the interactive session: Docker
Desktop (and with it the reverse proxy and every container), the media server, a hardware monitor. Windows Update
reboots outside the active hours, and after such a reboot most things stayed down until I logged in over RDP.

The previous workaround had been AutoLogon, and it had a nasty side effect: it got in the way of RDP and local
sign-ins. I did not want it back.

Acceptance criteria:

1. After a cold boot, **without anyone signing in**, everything is up within 10 minutes.
2. Signing in later does not start a second copy of anything, and signing out stops nothing.
3. Services that write to the media disk start only when *that* disk is mounted, not an empty folder with the same
   drive letter.
4. RDP and local sign-in keep working normally.

## Options

1. **AutoLogon.** Rejected (see above), and it keeps a password for automatic sign-in on the machine.
2. **Convert every app into a Windows service** with a service wrapper. Not all of them support it, and tray apps
   behave differently without a desktop.
3. **Scheduled tasks with an "At startup" trigger**, running in session 0, preferably with S4U (no stored password),
   with a stored password only where S4U does not work.

## Decision

Option 3.

- One task per app under a dedicated task folder, with a staggered delay after boot.
- A small wrapper script that every task calls: it waits until every required path exists **and** the volume's
  unique id matches the one recorded at install time, then starts the app; otherwise it gives up and logs why. This
  prevents the "the disk did not mount, so the app created a fresh, empty library on the system drive" failure.
- The per-user autostart entries were removed, so a later sign-in does not start duplicates.
- S4U where possible, including Docker Desktop. Later, the tasks of the non-admin service accounts
  ([ADR-007](007-per-service-accounts.md)) needed a stored random password, because S4U cannot be registered for a
  non-admin account; changing that password means updating the task with it, which a script does in one step.

## Verification

- A test script checks the expected state (processes, ports, containers, HTTP endpoints) and prints PASS/FAIL per item.
  Baseline before the change, signed in: 44/44. After the change: three reboots without signing in, 53/53 at the end.
- The same script runs after every planned reboot since then.
- Later, when the services moved to their own non-admin accounts ([ADR-007](007-per-service-accounts.md)), the same
  test proved that nothing regressed (71/0 after the reboot).

## Consequences

- An unattended reboot is a non-event for the family.
- Startup logic lives in scheduled tasks and a wrapper instead of in each app, which is one more thing to back up
  (the task definitions are exported nightly) and to document.
- Session 0 has side effects that only show up later: on this laptop the fan curve is driven by the vendor's
  desktop app, which needs a signed-in session. That became an incident of its own (see the case study).
