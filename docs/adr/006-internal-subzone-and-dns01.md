# ADR-006: Internal DNS subzone and ACME DNS-01 certificates with a single-zone token

- **Status:** accepted
- **Date:** 2026-10-22
- **Decided by:** me

## Context

The services were reached through a dynamic DNS name from a free provider, with certificates from the same setup.
I wanted my own domain, real certificates for every service, and names that work the same at home and over the VPN.
The same domain also hosts things that must **not** depend on the home server: a small public website and the
mail records. If the server is down, those must still resolve.

## Options

1. **Run the whole domain on the local DNS server** (split horizon for the apex). Simple, but then every device on
   the LAN or VPN depends on the home server for the public website and for mail, and a mistake in the local zone
   hides the real records.
2. **Public DNS records pointing to private addresses.** Works without local DNS, but publishes the internal layout.
3. **A dedicated internal subzone** (`home.<domain>`): the local DNS server is authoritative only for that subzone;
   everything else is resolved publicly as usual. The VPN uses split DNS for that subzone only.

For certificates:

1. **HTTP-01 challenges.** Needs inbound port 80 from the internet; the services behind the reverse proxy are not
   meant to be public.
2. **DNS-01 challenges through the DNS provider's API.** No inbound ports; needs an API token on the server.

## Decision

- **Subzone:** option 3. The dashboard is `home.<domain>`, every service is `<service>.home.<domain>`. A local zone for
  the whole domain, a split-DNS rule or a resolver policy that covers the apex is explicitly forbidden in the runbook.
- **Certificates:** DNS-01 through the reverse proxy, with an API token that can edit DNS records of **this one zone
  and nothing else** (no account-level permissions). The token is stored on the server, readable only by the proxy's
  configuration, and kept in my password manager as well.
- The old dynamic DNS name was retired after the migration: the proxy configuration was cleaned up, the provider token
  revoked and the zone deleted. Backups of the old configuration were kept until the new one had run for a while.

## Verification

- The migration followed a phased plan with a rollback for every step and a test matrix (TLS, LAN, VPN, mobile
  network from outside); certificates were first requested from the ACME staging environment. The old name stayed
  in place until the new names had passed.
- Certificate expiry is a monitored check: warning under 14 days, error under 3 days or if the certificate is
  invalid. There was no certificate monitoring at all before this.

## Consequences

- The public part of the domain never depends on the home server.
- A leaked token can change the records of one zone. That is still serious (it could redirect the public website),
  so it is in the accepted-risk register with its mitigation: scope, storage, and the ability to revoke it in minutes.
- Internal names are not secret (certificate transparency logs publish them), so nothing relies on them being hidden.
