# ADR-009: Narrow push notifications + external dead man's switch; everything else in the log

- **Status:** accepted (replaces "alerts only in the central log, passive")
- **Date:** 2026-11-02
- **Decided by:** me

## Context

All findings went to the central log, and I looked at it now and then. That works for trends, not for "the media
server is down on a Friday evening" or "the backup has not run for two days". The other extreme, a notification for
every warning, would teach me to ignore notifications within a week. And there was a blind spot no in-house alerting
can cover: if the server itself is off or hung, nothing on it can send anything.

## Options

1. **Keep everything passive** in the log.
2. **E-mail or push for every WARN and ERROR.**
3. **Push only for a short, explicit list of critical events**, plus an **external** check that notices when the
   monitor stops reporting.

## Decision

Option 3.

- **Channel:** ntfy on the public server, with a long random topic that only I know (the topic is the secret; it is
  read from a file and never logged). It reaches the phone at night as well.
- **Critical, and only these:** a family-facing service is unreachable, local DNS is down, no successful backup for
  26 hours, a security indicator of compromise in a media server's logs, low disk space or a worsening disk health
  value. Everything else stays in the log and in the daily report.
- **Safeguards:** at most six messages an hour; a storm (more than three critical incidents within ten minutes)
  becomes one digest; one reminder after six hours for outages, a daily reminder for backup and disk problems; an
  incident that recovers sends one "recovered" message; nothing is sent during maintenance windows or while an
  incident is flapping.
- **Content:** the check's name, severity, duration and the incident id. No addresses, no paths, no log excerpts.
- **Dead man's switch:** the monitor pings healthchecks.io every five minutes, but only while its own self-check
  passes (recent probe results, writable database, outbox not stuck). After 15 minutes without a ping,
  healthchecks.io notifies me through the same ntfy topic. That covers a crashed monitor, a hung machine, a power
  cut and a broken network with one rule.
- The weekly report is not pushed; it is in the log and in a file.

## Verification

- The planner's rules (dedupe, reminders, storm digest, maintenance, flapping) are unit tested.
- Shadow mode first: the monitor writes the messages it would send into the outbox without sending them, and they
  are compared with what actually happened; sending is switched on only after that. The heartbeat went live first,
  because it cannot produce false alarms about services.

## Consequences

- A notification means "act now". Anything that does not deserve that is, by definition, not on the list.
- The list is a decision, not a default: adding a check to it is a change I make deliberately, with the reason.
- A public ntfy server and healthchecks.io are external dependencies. If they are down I lose alerting, not the
  services; the log still has everything.
