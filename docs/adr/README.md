# Architecture decision records

> **Draft.** Planned records; each one follows [the template](template.md).

| # | Decision | Status |
|---|---|---|
| [ADR-001](001-nightly-backup-without-downtime.md) | Nightly restic backup with VSS and online database backups, without stopping services | accepted |
| [ADR-002](002-media-manifest-instead-of-copy.md) | Media library: file list and hash manifest instead of a full copy | accepted |
| [ADR-003](003-restore-testing.md) | Restore testing: weekly sampled data restore + on-demand service restore in isolated containers, no VM | accepted |
| [ADR-004](004-offsite-without-stored-credentials.md) | Offsite copy: manual weekly upload, no cloud credentials stored on the host | accepted |
| [ADR-005](005-unattended-startup-without-autologon.md) | Unattended startup with scheduled tasks and a storage-readiness gate; AutoLogon rejected | accepted |
| [ADR-006](006-internal-subzone-and-dns01.md) | Internal DNS subzone and ACME DNS-01 certificates with a single-zone token | accepted |
| ADR-007 | Per-service non-admin accounts; virtual service account for the monitor | planned |
| ADR-008 | No LLM in operations: deterministic, rule-based monitoring | planned |
| ADR-009 | Narrow push notifications + external dead man's switch; everything else in the log | planned |
| ADR-010 | Register of accepted risks | planned |
