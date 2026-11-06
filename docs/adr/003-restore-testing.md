# ADR-003: Restore testing: weekly sampled data restore + on-demand service restore in isolated containers, no VM

- **Status:** accepted
- **Date:** 2026-10-21
- **Decided by:** the approach was proposed by the assistant; I decided that the service-level test runs on request,
  not on a schedule

## Context

A backup that has never been restored is a hope, not a backup. I wanted two kinds of evidence: that the repository
itself is intact and the files come back, and that a restored service actually *starts and works* with its restored
data. The obvious tool for the second one is a virtual machine, but the server is a laptop that already runs close to
its thermal and memory limits, and a VM would also have to replicate the Windows-specific parts of the setup.

## Options

1. **Trust the backup tool's own success message.** Says nothing about whether a restore works.
2. **Periodic full restore into a VM.** The most realistic test, and the most expensive one to build and run.
3. **Two levels without a VM:**
   - a weekly automated data restore into a sandbox folder, with a sample of the repository data verified;
   - a service restore test that starts each service from its restored data in an isolated test container.

## Decision

Option 3.

- **Weekly, automated (Sunday 08:00):** `restic check --read-data-subset=10%` (a random 10% of the pack files every
  time), then a restore of the latest snapshot into a sandbox, where every database copy is opened and checked with
  `PRAGMA integrity_check` and the key configuration files are parsed. The result is a JSON report and an event-log
  entry; a failure raises an incident in the monitor.
- **Service restore, on request:** for every service, a test container (`rt-*`) with `--network none` is started
  from the official image and the restored data, and the test checks something only a working service can answer:
  the API status endpoint returns "initialized", the login page loads, a restored library contains the expected
  number of items. Native Windows services are tested the same way with the vendor's official container image. At
  the end, every test container, volume and downloaded image is removed.

I chose to run the service-level test on request (after a big change, or before a risky one) rather than weekly: it
pulls images and loads the machine for several minutes, and the weekly data-level test already catches a damaged
repository.

## Verification

- First service restore run: **26/26 PASS** in one run, repeated a day later with the same result. Afterwards there
  were no leftover test containers or volumes, and every production container was still running.
- The weekly data-level test has passed in every run so far (25–28 checks, depending on the number of services at
  the time). It also caught gaps that a success message would not have: data that the test did not
  sample yet is listed explicitly as "not covered" in the report, and was covered by a one-off restore.
- The report distinguishes *full restore tested*, *database tested in sandbox* and *configuration only*, so it is
  visible which service has the weaker kind of evidence.

## Consequences

- There is a measured answer to "can we restore X?" for every service, without a VM.
- The recovery *time* is not measured yet: the tests prove that a restore works, not how long a full disaster
  recovery takes. A bare-metal test in a VM stays an optional later step.
- `--network none` keeps the test containers away from production and the internet, but it also means a service is
  only checked up to what it can do offline.
