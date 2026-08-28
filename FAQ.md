# Agent Monitor for BES Clients: Frequently Asked Questions

Answers to common questions about what Agent Monitor is, how it works, and how it keeps your data and your endpoints secure. If you don't find your question here, you're welcome to open an issue on the project's GitHub repository.

## Contents

1. [Getting Started: Installation and Deployment](#1-getting-started-installation-and-deployment)
2. [What It Monitors: The Health Model](#2-what-it-monitors-the-health-model)
3. [Networking and Forwarders](#3-networking-and-forwarders)
4. [Authentication and Access Control](#4-authentication-and-access-control)
5. [Enrollment and Agent Identity](#5-enrollment-and-agent-identity)
6. [Encryption and Report Confidentiality (Strongbox)](#6-encryption-and-report-confidentiality-strongbox)
7. [Code Signing and Build Attestation](#7-code-signing-and-build-attestation)
8. [Certificates and TLS Trust](#8-certificates-and-tls-trust)
9. [Tenants, Alerts, and Notifications](#9-tenants-alerts-and-notifications)
10. [Settings and Configuration](#10-settings-and-configuration)
11. [Security Assurance and Review](#11-security-assurance-and-review)
12. [Migration, Backup, and Operations](#12-migration-backup-and-operations)

---

## 1. Getting Started: Installation and Deployment

### What do I need to deploy Agent Monitor?
Three pieces, two of them required:

- **The Dashboard:** a self-hosted web application on a single Ubuntu Server (24.04 or 26.04 LTS). This is where everything is managed.
- **The Health Agent:** a lightweight agent that runs on each endpoint you want to monitor, Windows or Linux.
- **The Forwarder (optional):** a relay for endpoints that cannot reach (or are not allowed to contact) the dashboard directly, such as machines in a restricted network or machines outside your network.

### How do I install the dashboard?
Extract the tarball and run the installer on a clean Ubuntu server. Everything it needs is installed and configured by the installer (web server, application service, and PostgreSQL database). Internet access from the server is required at this step so the components can be downloaded. For the full step-by-step walkthrough, see the [Installation Guide](docs/AgentMonitor-Installation-Guide.pdf).

### How does the Health Agent get onto endpoints? By Fixlet or manually?
Both are supported. The installer gets its settings either through explicit command-line values or a charter file that you can place in the same folder as the installer. If neither is provided, it also has an interactive prompt.

- **Through a Fixlet:** it downloads the agent package and a small charter file, drops the charter beside the installer, and runs the installer silently. The charter carries everything the agent needs to configure itself.
- **Manually:** an admin user runs the installer directly on a device currently running a BES Client.

### How do I set up a forwarder?
1. Create the forwarder in the dashboard first. This gives you the forwarder's identity key.
2. Install the forwarder software on the host, passing it the dashboard URL and that forwarder key.
3. The forwarder then contacts the dashboard, registers itself, and enables automatically once the two are talking.
4. After it registers, set its Placement (Internal or DMZ) from the Forwarders table.

### When I install a forwarder, do I need the charter file too, or just the setup program?
Just the setup program. The forwarder authenticates with its own forwarder key (the one shown when you create it in the dashboard), not with a charter.

---

## 2. What It Monitors: The Health Model

### What does the agent check to decide an endpoint is Unhealthy?
It watches for three things:

1. Is the BES Client service running?
2. Is the BES Client still writing to its log recently? (checked against a freshness window, 2 hours by default)
3. Is the client's core configuration file (`actionsite.afxm`) present?

An endpoint is Unhealthy if the service is stopped, or there is no log at all, or the log has gone untouched for more than four times the freshness window (8 hours by default), or the core file is missing. It is Healthy when the client is running and the log is fresh. In the rare case where the agent cannot even locate the BigFix data directory, the status is Unknown.

### How often does the agent check?
There are three layers working together:

1. The agent is launched by the operating system scheduler every 10 minutes on Linux and every 15 minutes on Windows.
2. It runs a full self-check about every 10 minutes. If an endpoint is unhealthy, the next check is pulled in sooner so recovery is noticed quickly.
3. A watchdog catches a sudden BES Client failure in real time. On Linux this is event-driven and reacts in seconds. On Windows it polls the service, every 60 seconds by default (adjustable to 1, 2, 5, or 10 minutes in System Settings).

### If nothing is wrong, does the agent still report?
No, with a caveat. The agent uses "exception-first reporting." That is, it reports immediately when something changes (goes unhealthy, recovers, first installs, is uninstalled, renamed, or upgraded) and stays quiet otherwise, except for a periodic heartbeat about every 22 hours. It also sends one extra heartbeat right after a reboot so a rebooted machine does not look falsely silent.

Every check-in, including a plain heartbeat, updates the endpoint's last-seen time on the dashboard and keeps it in the healthy bucket. As long as you see a check-in within the last 24 hours, that is your proof the agent is installed and monitoring.

### What does the agent send when it finds a problem?
An event report identifying the machine and explaining the problem. It carries the essentials: hostname, OS and agent version, the status and the reasons for the status, and what the agent observed about the BES Client (running or not, version, when the log last changed, when it last reported, etc.).

On the moment an endpoint first becomes unhealthy, the agent also attaches a one-time diagnostic bundle for troubleshooting. This includes a tail of the client log plus network state, clock offset, free disk space, select client settings, and recent log errors. This bundle is sent only on the transition into unhealthy, not on every report.

### Does the Health Agent take any action to fix a broken BES client, such as restarting it?
No, and that's intentional.

The Health Agent is deliberately report-only. It watches and reports. It doesn't start, stop, restart, or modify the BES client, and it has no ability to make any changes to the rest of the endpoint. I chose to limit the Health Agent's boundary to reporting for two reasons:

1. Keeps its footprint small and its required privileges low.
2. Its behavior is easy to trust and whitelist because it only observes and reports.

When the BES client stops or hangs, Agent Monitor's job is to make that visible right away (as Unhealthy or Silent) and alert you, so the failure doesn't stay hidden. Detection and remediation stay cleanly separated. Agent Monitor tells you what's wrong. You decide how to fix it.

### How can you ensure that the report is not intercepted by a middleman that can read vital information about the endpoint?
If report encryption is enabled for the tenant, the whole payload is encrypted before it leaves the machine. The diagnostic bundle is encrypted using a hybrid X25519 + ML-KEM-768 key encapsulation (a classical elliptic-curve key exchange paired with a post-quantum one), HKDF-SHA256 to derive the encryption key, and AES-256-GCM to encrypt the payload. Using a classical and a post-quantum method together means the protection holds even if either one is broken later, including by a future quantum computer.

### What is the difference between Unhealthy, Silent, and Disagreement?
- **Unhealthy:** The agent's own checks say the BES Client is broken.
- **Silent:** The agent itself has stopped checking in. This could simply mean that the machine may be off, cut off from the network, or the health agent itself may be broken.
- **Disagreement:** The agent is Silent for too long.

### How long before an endpoint is Silent, and before it becomes a Disagreement?
An endpoint becomes Silent after 24 hours with no check-in. After 72 hours of being silent, the dashboard cross-checks against BigFix's own records to see if the device has posted a "Last Report Time" more recent than the silent period. If BigFix says the machine reported recently, the two sources of truth disagree. This usually points at the agent (or its path to the dashboard) rather than the machine being down, so it is flagged as its own higher-priority state instead of falsely calling the machine unhealthy.

---

## 3. Networking and Forwarders

### If an agent has no direct path to the dashboard, can it still report?
Yes, through a forwarder. The agent reports to a forwarder that does have a path to the dashboard. The forwarder passes the report along.

### How does the forwarder work?
It is a store-and-forward "relay." It serves only the agent reporting paths and passes each request through untouched, adding only its own identifying header. If the dashboard is briefly unreachable, the forwarder holds the reports and replays them in order once the dashboard returns, answering the agent with an "accepted" in the meantime. It never decodes or reads the reports. It moves the exact bytes through in each direction.

### Do agents always go through a forwarder, or only when they need to?
Agents talk to the dashboard directly whenever they can. A forwarder is used when an endpoint cannot, or is not allowed to, reach the dashboard directly. Each forwarder is marked Internal or DMZ. If an administrator manually assigns agents to a forwarder, those agents will prefer an internal forwarder while on the corporate network and a DMZ forwarder when off it. A per-group policy can also prevent a group of endpoints from ever using an external forwarder.

### Is the agent to forwarder to dashboard path secure?
Yes.

- Both hops run over TLS/HTTPS, so traffic is encrypted on the wire.
- The agent verifies the forwarder's identity from a certificate issued by an authority the agent trusts.
- The forwarder authenticates itself to the dashboard with its own key, which you can disable or revoke instantly.
- When report encryption is enabled, the report itself is encrypted end to end.

### If someone compromised a forwarder, could they read the diagnostic bundles?
With report encryption enabled, no. The report body is a sealed "strongbox" that only the dashboard can open, so a compromised forwarder sees only a locked envelope and some metadata (who talked to the dashboard, when, and from what address), never the contents. And regardless of encryption, every report is signed by the agent, so a compromised forwarder can never forge or alter a report, and its own credential can be revoked at any time. (If report encryption is off for a tenant, the body would be visible to a compromised forwarder, which is exactly why encryption exists for untrusted-network path scenarios.)

---

## 4. Authentication and Access Control

### How is the login protected against common attacks?
The dashboard follows standard secure-development practices. User input is always handled so that classes of attack such as SQL injection cannot reach the database, the application runs on a memory-safe platform, and repeated login attempts are bounded by rate limiting and automatic account lockout. Passwords are never stored in readable form, and administrators can enforce a password policy for length, complexity, expiry, and reuse.

### Can a user see data outside their scope, gain privileges they were not given, or act as another user?
No. Access is multi-tenant and deny-by-default. A user sees only the tenants granted to them, and that boundary is enforced on the server for every request, not in the browser. Roles are checked on the server for every action, and a session token cannot be altered to grant privileges the account does not actually have. Sensitive administrative actions are recorded in the audit log.

### Is there multi-factor authentication (MFA)?
Not yet. Authenticator-app based MFA is on the roadmap. Today, access is protected by the password policy, account lockout, and rate limiting, and certain sensitive actions require you to re-enter your password before they proceed.

### Can I turn off the password reset email?
The reset-link email is the recovery mechanism itself, so it always sends while self-service reset is enabled. There is no per-user switch to disable it. An administrator can disable self-service reset globally, in which case admins reset passwords directly from User Management. The separate "password changed" security notice is a normal notification you can control.

---

## 5. Enrollment and Agent Identity

### What does enrollment mode "auto" mean?
It is a per-tenant setting that controls what happens when a new endpoint enrolls into that tenant:

- **auto (the default):** the endpoint is approved automatically and starts reporting right away, with no operator step.
- **admin approval:** a new endpoint lands in a pending state and does not report until an administrator approves it.

Either way, which tenant an endpoint belongs to is decided automatically from the machine's BigFix gather URL.

### What stops a rogue device from faking a heartbeat or impersonating an endpoint?
Every agent has its own private key, generated on that endpoint and never shared. It signs every message it sends, and the dashboard verifies each one against that endpoint's enrolled key, confirms the message is recent and not a replay, and confirms the endpoint is enrolled and not revoked. A rogue device does not hold the private key, so it cannot produce a valid signature, and unsigned requests are always rejected.

### If I revoke an agent, can it re-enroll itself, or must I reinstall?
A revoked agent can't quietly re-enroll itself. That's the point of revocation. There are two recovery paths:

- **Ordinary revoke:** an administrator can Restore the endpoint from the dashboard, which reactivates it with its same identity.
- **Permanent revoke:** if you escalated to a permanent revocation, even Restore is refused, and the only way back is a fresh reinstall, which creates a brand-new identity.

### What are re-enrollment grants?
When an endpoint's enrollment is revoked, its reports stay rejected until an administrator authorizes it to enroll again. That authorization comes in three types: a single endpoint, an entire tenant (which requires a second administrator to approve), or global as a last resort. Each type is time-limited, and when more than one applies, the narrowest one wins.

### Why does a reinstalled or wiped machine show up as a new entry?
The dashboard identifies an endpoint by its cryptographic identity, not its hostname. A reinstall (or a wipe-and-reinstall) generates a new identity, so the machine appears as a new entry. The old entry goes idle. Nothing reports as that identity anymore, so it simply goes Silent, and you can hide it (a reversible soft-retire that removes it from all views and alerts). Hidden entries are purged automatically over time. Note that per-endpoint settings such as the Critical flag and group membership (which carries alert routing) belong to the identity, so you have to re-apply those on the new endpoint id.

---

## 6. Encryption and Report Confidentiality (Strongbox)

### What is a "Strongbox"?
Strongbox is the report-encryption subsystem, the sealed box a report travels in. Each tenant has its own encryption keypair. The dashboard holds the private half, and agents hold only the public half. That means an agent can encrypt a report so that only the dashboard can open it, and nothing in between (the network or a forwarder) can read it. Strongbox is opt-in per tenant, with three modes (off, optional, required), and it ships off by default. When a tenant is set to optional or required, that tenant's reports are encrypted end to end. When it is off, reports are protected only by the transport (TLS).

### Which encryption and signing algorithms does Agent Monitor use?
- **Signatures (authenticity):** every agent report is signed. New agents use ML-DSA-65, a post-quantum signature standard (NIST FIPS 204), designed to stay secure even against future quantum computers.
- **Encryption (confidentiality):** when Strongbox is enabled, reports are encrypted with a hybrid key exchange, X25519 combined with ML-KEM-768. The shared secret is run through HKDF-SHA256 to derive a key, which drives AES-256-GCM to encrypt the message.

Using a classical and a post-quantum method together protects against both today's threats and "collect now, decrypt later" threats.

### How do agents get the encryption key, and can it be changed?
Each tenant's public encryption key is delivered to its agents over their normal authenticated channel, and it is signed so an agent can confirm the key is genuine before trusting it. At install the agent is given the dashboard's identity, so it can confirm each key was signed by that dashboard. Keys can be changed at any time. An administrator can rotate a tenant's key routinely, or immediately invalidate the keys as part of an incident response. Either way, agents pick up the new key automatically the next time they check in, with no work on the endpoints.

---

## 7. Code Signing and Build Attestation

### How does the dashboard know an agent build is legitimate?
Every agent package carries a small signed statement naming the build's version, platform, and a fingerprint (a SHA-256 hash) of the exact binary. That statement is signed at release time with a private key that exists only on the developer's release systems and never ships in any product. The dashboard carries the matching public key built into its own code, and a public key can check a signature but can't create one, so it's safe to distribute.

When an agent reports in, the dashboard checks two things. That the statement was really signed by the developer's key, and that the fingerprint in the statement matches the fingerprint the agent measured on its own binary. If they match, the build is recorded as known-good automatically. This works entirely offline. The dashboard never has to contact the developer to confirm.

### Can an attacker substitute a fake or tampered agent and have it accepted as genuine?
No.

- A statement signed with the attacker's key is checked against the developer's public key and rejected. The key that decides whether a statement is legitimate is built into the dashboard.
- A statement stolen from a real package and attached to a tampered binary fails too. The statement names the fingerprint of the real binary, which no longer matches the tampered one.
- A binary with no statement at all is never silently trusted.

### Why is the Windows agent signed with a code-signing certificate?
On Windows, executables carry an embedded code-signing signature that the operating system itself can verify, so signing the Windows agent lets your Windows endpoints confirm the agent is the one you approved.

### Why do I need to own that certificate?
Because if I as the developer owned that certificate then I could create any software that your endpoints would implicitly trust. By owning the signing certificate yourself, no one outside your organization holds a key your endpoints trust, and you can replace the certificate at any time. The Linux agent is verified through the build attestation above and needs no signing.

### Do I absolutely have to generate a signing certificate in the dashboard?
No. If you already have a code signing certificate that you own, you can use it to sign the Windows agents. Note: a BYO certificate does need to be one your Windows endpoints already trust, meaning it chains to a CA in their trust store. This is normally the case for a cert from your internal PKI or a public CA.

### Why is there a "download unsigned" option?
Because signing is yours to control. The unsigned download lets you take the agent and sign it with your own tooling or certificate outside the dashboard, or deploy it as-is. The dashboard offers to sign for you, but it never forces the workflow.

### Can I run the agent 'unsigned'?
Yes. Just be aware that doing this may cause Windows Smartscreen to warn users before the executable runs and if your organization uses application control software like Applocker, it may block unsigned or unauthorized executables from running.

### Can a rogue administrator import a bad attestation to whitelist a malicious agent?
No. The set of trusted developer keys is compiled into the dashboard. No screen, API, or file can add to it. The Import function on the Agent Builds page accepts only statements that are already signed by the developer, and the dashboard verifies that signature before writing anything.

### Why do Linux builds show no thumbprint?
The thumbprint is the fingerprint of a Windows code-signing certificate, which is a Windows-only mechanism. Linux has no equivalent OS signature to fingerprint, so the column is empty by nature, not by omission. Both platforms still carry the developer-signed build attestation.

---

## 8. Certificates and TLS Trust

### Does the dashboard need a certificate from a public certificate authority?
No. The dashboard can run its own internal certificate authority and issue its own TLS certificate automatically, so HTTPS works out of the box with nothing to buy, but you can use your own certificate or a CA certificate you've bought instead.

### The dashboard runs its own internal certificate authority. Is that a weakness an auditor will flag?
Not necessarily. Private, internal CAs are standard enterprise practice, the same pattern used by tools like Active Directory Certificate Services and HashiCorp Vault. Auditors do not flag "you run a private CA." They look at how the CA key is protected, scoped, and managed. In this particular case:

- The CA private key is stored so the web application can't read it. It's used only to sign, and the web server only ever presents the issued certificate.
- The CA is name-constrained to this deployment, so even a leaked CA key can't issue a valid certificate for an outside domain, and it can't create sub-authorities.
- Trust is narrow: only agents configured to trust this one deployment use it, and it's never pushed into operating-system or browser root stores.
- Rotation is non-breaking: reissuing a certificate doesn't disturb agents, because they trust the authority, not the individual certificate.

### How does an agent decide whether to trust the dashboard's or a forwarder's certificate?
When an agent opens an HTTPS connection, the server presents a certificate. The agent checks two things, both of which must be true:

1. Was the certificate issued by an authority the agent trusts? Each agent carries a short list of trusted authorities, and the certificate's signature is checked against it.
2. Does the name on the certificate match the address the agent dialed? If the agent connected to a name the certificate doesn't list, the connection is refused even if the signature is valid.

Agents trust the authority, not individual servers, so once an agent trusts the authority it trusts every certificate that authority issues, including forwarders you stand up later.

### Reports use post-quantum cryptography. Why is the TLS certificate still a classical algorithm?
Because a TLS certificate is checked by the standard web-security stack built into the OS and libraries, and those stacks can't yet verify post-quantum certs. This matters less in the "save now, decrypt later" threat model because a TLS certificate is a signature that only matters live during the handshake, so it can't be recorded now and broken later. On top of that, the report contents themselves are already post-quantum protected, so the data that actually matters is covered regardless.

### Can agents connect by IP address, or do they need a DNS name?
Either works, as long as the address the agent uses is one the certificate was issued for. A certificate can list both DNS names and IP addresses, and the agent checks that the address it dialed is on the certificate. So you can point agents at a name or an IP, provided that name or IP is covered by the certificate.

---

## 9. Tenants, Alerts, and Notifications

### When does the dashboard send an alert email?
The dashboard emails an alert when a critical endpoint turns unhealthy. A non-critical endpoint that turns unhealthy still shows as Unhealthy on the dashboard, but it does NOT send an email, so alerts stay focused on the machines you've marked as the most important. Alerts are routed per group, each group has its own recipients, and they're bundled per recipient so a burst of failures arrives as one combined email rather than a storm.

### Why does an endpoint sometimes take a few minutes to show as unhealthy on the dashboard?
When a non-critical endpoint changes state, its agent holds the report for a small randomized delay (up to 10 minutes under normal load) before sending it, so that thousands of endpoints recovering from a shared outage don't all hit the dashboard with reports at the same time. That's why a non-critical endpoint's status can lag briefly. Critical endpoints skip this delay entirely. They report immediately, so their status, and their alert email, go out right away. If a machine you expected to be immediate seems to lag, check that its current record actually carries the Critical flag (a reinstalled machine is a new record and needs the flag re-applied).

### Do I get alerted when an endpoint goes Silent or into Disagreement?
No. Email alerts fire only when a critical endpoint turns unhealthy. Silent and Disagreement are shown on the dashboard but don't send email, because a Silent endpoint is often just powered off and a Disagreement usually resolves itself. This keeps alert emails meaningful rather than noisy.

### Can I stop alerts during planned maintenance?
Yes. Schedule a maintenance window for the endpoints, group, or tenant you're working on, and the dashboard suppresses their alerts for that period, then resumes normal alerting automatically when the window ends. Health changes are still recorded during the window. They just don't raise alerts, so planned reboots and patching don't set off a storm. Endpoints in a window show a Maintenance label.

### Why did an alert email show a tenant name I didn't set, and can I rename a tenant?
There's no global "rename tenant." A tenant's stored name comes from its BigFix masthead. What you can set is a per-user alias. Each dashboard account can label tenants its own way, and every screen shows your alias. Because alert emails go to specific recipients, an email uses the alias belonging to the account that owns that recipient's address. A recipient address with no matching account sees the stored name. So if one person has two accounts, set the alias on both to keep it consistent.

---

## 10. Settings and Configuration

### What does "Windows BES Client Poll Interval" mean, and why is it Windows-only?
The agent runs a watchdog that watches the BES Client service so it reacts quickly when the service stops, starts, or crashes, instead of waiting for the next scheduled self-check. On Windows, the watchdog detects those changes by polling the service, and this setting controls how often it checks (60 seconds by default). Lower means faster detection with slightly more frequent checks. Higher means a lighter footprint and slightly slower detection. Linux doesn't need the setting because its watchdog is event-driven so there's no interval to tune. A change takes effect within one cycle, with no restart.

### Why is there a maximum on the log size the agent sends?
The Max Log Tail Size setting caps how much log an agent attaches to a diagnostic bundle (2 MB). The cap is enforced in the interface, on save, and again at the server. It's a deliberate boundary. An agent trying to submit more log than the ceiling usually points to something worth investigating rather than something to accommodate, and it keeps a single report from ballooning plus controlling database usage.

### What is "Signature Clock Skew"?
Every signed message the agent sends includes a timestamp, and the dashboard rejects any message whose timestamp is too far from its own clock. This is a safety feature that stops an attacker from replaying an old captured message. "Clock skew" is when an endpoint's clock is set wrong. The allowed difference is about five minutes (tunable). If a machine's clock drifts beyond that, its messages look stale or from the future even though the signature is perfectly valid, so the dashboard rejects them and the endpoint appears to stop reporting. The fix is simply to correct the machine's time sync. It's a common cause of an otherwise healthy agent failing to report.

---

## 11. Security Assurance and Review

### Is the security of the platform reviewed?
Security is treated as an ongoing practice, not a one-time event. The platform is built to standard secure-development practices, and its security is reviewed as the code evolves, with each significant release reviewed again for what changed. As with any software you bring into a sensitive environment, I always encourage running your own review before relying on it for high-stakes protection.

### Can I trust the encryption?
The encryption is assembled from standard cryptographic building blocks, combining a classical method with a quantum-resistant one, and the construction is reviewed before the feature is enabled. The building blocks themselves are the widely vetted, standardized ones. Because this is free software, there's no paid third-party cryptographic certification, so for high-stakes use I'd recommend your own cryptographic review as well.

### Where can I get more detailed security information?
The materials here name the protections in place. For a deeper review, including the trust model and assessment details, contact me via the Github repo for the security documentation, which is shared through that channel rather than posted publicly.

---

## 12. Migration, Backup, and Operations

### Does the dashboard have backup and restore?
Yes. You can back up and restore the database from the web interface. A backup captures your endpoints, groups, settings, and history, so you can recover the dashboard or move it to new hardware.

### How is a migration between servers protected?
When you move a dashboard to another server, the export file is both encrypted and signed. It's encrypted with a passphrase you choose (AES-256-GCM), so without the passphrase its contents are unreadable, and it carries a signature that the receiving server verifies before it even attempts to decrypt. A single altered byte, or a file that isn't a genuine export, is refused up front. So someone intercepting the file can neither read it nor slip in a modified one.

### What does the Server Status indicator (top-right) show?
It's a live load light for the dashboard server itself, not an endpoint status. It shows how busy the dashboard's database capacity is: green when there's plenty of headroom, yellow as it gets busier, and red when it's heavily loaded. It refreshes every 30 seconds, and clicking it shows the actual numbers. The same signal is also used to gently stagger agent reporting when the server is busy, so load smooths out on its own.

### Does the dashboard send any data back to the developer?
No. The dashboard is entirely self-hosted, with no cloud services and no telemetry back to the developer. All trust decisions, including verifying agent builds, are made locally and work offline or air-gapped. The only reason the server would need to have internet access is to download the necessary component packages (PostgreSQL, Nginx, Ubuntu, etc.) and updates.
