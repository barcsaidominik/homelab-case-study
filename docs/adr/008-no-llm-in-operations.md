# ADR-008: No LLM in operations: deterministic, rule-based monitoring

- **Status:** accepted
- **Date:** 2026-11-02
- **Decided by:** me, after a read-only assessment of the options with the assistant

## Context

Until then the server was watched by a scheduled health-check script every six hours that wrote its findings to the
central log. I read the log when I remembered to. I had been building the whole setup with an AI coding assistant, so
the obvious next idea was to let an AI watch it too: read the logs, decide what is wrong, maybe fix it.

## Options

1. **A cloud LLM that receives the logs and health data and diagnoses problems.**
2. **An AI agent with remote access to the server** that can investigate and act, around the clock.
3. **A local LLM on the server.**
4. **A deterministic monitor:** fixed rules, an incident state machine, a narrow set of notifications.

## Decision

Option 4, built as [homelab-monitor](https://github.com/barcsaidominik/homelab-monitor). The others were rejected for
concrete reasons:

- **Cloud LLM:** the state of the house (which services exist, when people are home, what fails) would leave the
  house continuously. And a diagnosis that is not reproducible cannot be tested.
- **Remote agent:** to be useful it would need admin access to the server, around the clock. The only safe version
  would be so restricted that it could do little more than the rules can. An agent that can act on the server is
  exactly the kind of thing the rest of the design tries to keep away from it.
- **Local LLM:** the server's GPU (6 GB) is shared with hardware video transcoding, and the laptop's cooling is
  already at its limit under load. The machine with the large GPU is my desktop, which is not on around the clock.
- **Rules are enough here.** The questions are boring and well-defined: is this endpoint up, did the backup run, is
  the disk filling up, is the fan spinning. What made alerting noisy was not a lack of intelligence but missing
  hysteresis and dependencies, and those are rules.

## Verification

- Every rule (thresholds, suppression, flapping, maintenance windows, notification planning) is covered by unit tests
  with an injected clock; the same input always gives the same decision.
- The monitor runs in shadow mode first (notifications written to the outbox but not sent), so what it *would*
  have sent can be compared with what actually happened before anything is sent.

## Consequences

- No data leaves the house except the notifications themselves, which contain only names, severity, duration and an
  incident id ([ADR-009](009-narrow-notifications-and-heartbeat.md)).
- The monitor does not "understand" anything new: a failure mode nobody wrote a check for stays invisible until a
  check is added. Incidents are reviewed with that in mind.
- AI stays where it is useful for me: as a pair programmer for writing and reviewing the code and the documents, with
  every change reviewed and every claim backed by evidence, not as an operator with access.
