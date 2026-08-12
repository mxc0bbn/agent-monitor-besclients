# Changelog

All notable changes to Agent Monitor for BES Clients are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/).

For full release notes (upgrade instructions, assets, checksums) see the
[Releases page](https://github.com/mxc0bbn/agent-monitor-besclients/releases).

## [v2.7.2.2] - 2026-08-12

### Fixed
- **Windows agents now trust a private-certificate dashboard or forwarder on
  their own, with no manual certificate step.** When a Windows endpoint installs
  against a dashboard or forwarder that uses its own certificate authority, the
  installer captures the complete certificate chain during setup, so certificate
  verification turns on by itself and the agent reports over a verified
  connection. Previously only part of the certificate was captured on Windows,
  so verification could not be enabled without delivering a certificate by hand.
  Windows only; the Linux agent, dashboard, and forwarder are unchanged from the
  previous release.

## [v2.7.2.1] - 2026-08-10

### Fixed
- **Health Agent upgrades no longer risk losing the dashboard trust settings.**
  On dashboards that use a private certificate authority, an earlier build could
  stop reporting (show as Silent) after an upgrade. The agent now preserves its
  established trust across upgrades and never writes an unusable trust setting.

## [v2.7.2] - 2026-08-05

### Changed
- **The agent finds BigFix wherever it is installed.** If the BES Client, Root
  Server, or Relay is on a non-default drive or folder, the agent still locates
  it, so enrollment and Root Server / Relay detection work on any install layout.
  Previously it looked only in the default locations.
- **Connect by name or IP, with full certificate verification and no manual
  certificate steps.** The dashboard's certificate now covers every way an agent
  might address it (the reporting name you choose, the host name, localhost, and
  the IP), so verification passes whether an agent points at a name or an IP.
  Because the certificate is issued by the dashboard's own certificate authority,
  changing or adding a reporting name can be reissued without touching every agent.
- **Path settings accept quotes.** A folder path in the configuration works with
  or without surrounding quotation marks.

### Security
- The dashboard's certificate authority key is stored so only the administrator
  account can read it, and it is constrained to vouch only for that one dashboard.

## [v2.7.1] - 2026-07-30

### Changed
- Maintenance and environment-wide rollout release. No functional changes to the
  Health Agent.

## [v2.7.0] - 2026-07-27

### Changed
- **TLS verification is now on by default.** The Health Agent authenticates the
  dashboard's certificate before sending anything, and holds its reports rather
  than send to an unverified peer.

### Security
- Additional hardening: credentials are never sent across a redirect to another
  host; trust and routing changes are accepted only directly from the dashboard,
  never through a relay; server-pushed timing is capped so the agent cannot be
  told to effectively stop reporting; and key rotations require a signed
  confirmation before taking effect.

## [v2.6.7] - 2026-07-21

Initial public release.

### Added
- **Out-of-band health monitoring** for BigFix (BES) client endpoints. A
  lightweight agent on each endpoint reports independently of the BES client,
  so a broken client is distinguished from one that is merely powered off,
  isolated, or slow to check in.
- **Heartbeat and diagnostic reporting.** Every agent run sends a small
  heartbeat; a richer diagnostic bundle is sent only when the agent's local
  verdict is unhealthy. A missing heartbeat is itself a signal, surfaced as a
  Silent endpoint.
- **Dashboard** with Unhealthy, Silent, and Disagreement views, endpoint
  browsing, and per-endpoint detail with trend history.
- **Alerting** for critical endpoints that turn unhealthy, with per-group
  recipients and per-recipient email bundling.
- **Endpoint groups** with criticality, per-criticality self-check intervals,
  and group-scoped alert routing.
- **Cross-platform Health Agent** for Windows and Linux, deployable through the
  BigFix Console.
- **DMZ forwarder** for relaying agent traffic from isolated network segments to
  the dashboard over a dedicated machine-plane port.
- **Charter-based enrollment.** Endpoints enroll against a signed charter with
  per-endpoint keys; enrollment fails closed if the charter cannot be verified.
- **Post-quantum signing** (FIPS 204 ML-DSA) for build attestation and agent
  report signatures.
- **Role-based access with tenant scoping.** Non-administrative users are scoped
  to their tenant with deny-by-default access.
- **Security hardening.** Password policies, account lockout, encrypted
  credential storage, TLS by default, and a startup integrity manifest over the
  distributed application files.
- **Administration.** Audit log with CSV and PDF export, backup and restore,
  email/SMTP configuration, API key management, and per-page PDF and CSV export.
- **Dark and light themes.**
- **Guided installer** with live validation of the dashboard URL and enrollment
  key.
