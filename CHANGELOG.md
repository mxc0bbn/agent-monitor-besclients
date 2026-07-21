# Changelog

All notable changes to Agent Monitor for BES Clients are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/).

For full release notes (upgrade instructions, assets, checksums) see the
[Releases page](https://github.com/mxc0bbn/agent-monitor-besclients/releases).

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
