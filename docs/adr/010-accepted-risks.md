# ADR-010: Register of accepted risks

- **Status:** accepted, reviewed when something in the setup changes
- **Date:** 2026-10-21 (started), last updated 2026-11-06
- **Decided by:** me, one entry at a time; the assistant points out the risk, I decide whether to fix or accept it

## Context

A home server never reaches "fully secure and fully redundant". Every security assessment produced findings that
could be fixed only at a price: new hardware, a worse experience for the family, or a lot of work for a small gain.
Leaving them undocumented has two bad outcomes: they get forgotten, or they get "fixed" by someone (me, half a year
later, or an assistant) who no longer knows why they were left that way.

## Decision

Every risk that is not fixed is written down with the reason, the mitigation that *is* in place, and the condition
that would reopen it. The rule for assistants working on the setup: an accepted risk is not proposed again unless
its reopening condition is met.

| Risk | Why it is accepted | What limits it | Reopen when |
|---|---|---|---|
| Container ports listen on all interfaces, so the reverse proxy can be bypassed on the LAN | Several devices reach services only by IP and port | Host firewall: LAN and VPN only, nothing exposed to the internet | A device that needs IP:port goes away, or a service gets internet exposure |
| The central log accepts events without authentication | Every script and app would need its own key | Web UI needs a login; reading needs a read-only key; reachable from the LAN and VPN only | Anything outside the host needs to send logs |
| The monitor's and a local app's read-only APIs have no authentication | They return aggregates the dashboard needs; a key in the dashboard config would not add protection | GET only, no secrets or personal data, LAN and VPN only | An endpoint starts returning anything sensitive |
| One DNS server: if the host is down, local name resolution is down | A second device for DNS was not wanted at the time | A watchdog restarts the DNS service; DNS is a critical, notifying check; clients can be switched to public DNS by hand (internal names then stop working) | A spare device becomes available (under consideration) |
| The backup and the media library are on the same physical disk | No second large disk | Weekly offsite copy of the backup ([ADR-004](004-offsite-without-stored-credentials.md)); the media is re-obtainable ([ADR-002](002-media-manifest-instead-of-copy.md)) | Disk health degrades, or the backup outgrows the offsite plan |
| The offsite copy is manual, so its age depends on me | Automation would put cloud credentials on the server | Weekly reminder; upload right after big changes | The repository grows beyond what a manual upload can handle |
| The DNS API token on the server can change the records of the public zone | DNS-01 certificates need it | Scoped to one zone; readable only by the proxy's configuration; revocable in minutes; certificate and record changes are visible | The provider offers record-level token scopes |
| A media server's remote connections prefer, but do not require, encryption | Several internal consumers talk to it over plain HTTP on localhost | Remote playback is checked to be encrypted; the health check warns if it is not | All internal consumers can use the encrypted endpoint |
| The services' scheduled tasks hold stored random passwords | S4U cannot be registered for non-admin accounts | Passwords are random, never typed, rotated only by script; accounts cannot sign in interactively | Windows offers a password-less option for these accounts |

## Verification

The register is checked whenever a security assessment runs or the setup changes; each row points to the decision
or the assessment it came from. An entry leaves the register only by being fixed (with evidence) or by its reason
disappearing.

## Consequences

- Nobody re-litigates the same trade-off every month, and nobody silently undoes it either.
- The register is also the honest answer to "how secure is it?": what is protected, what is not, and why.
