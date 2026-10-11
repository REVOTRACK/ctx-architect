# CORTEX_KNOWN_UNKNOWNS.md
**Registry of Unresolved Questions, Binary Contracts & Confidence Labels**  
**Audit Date:** 2026-10-11  
**Operating Mode:** Read-Only Verified Audit

---

## 1. Classification Methodology

Every claim in the Cortex audit is classified according to four strict confidence standards:
- **[VERIFIED FACT]:** Confirmed directly by reading source code, running read-only inspection commands, and tracing callers/consumers.
- **[LIKELY INTERPRETATION]:** Highly consistent with observable code and configuration, but missing a closed-loop runtime confirmation.
- **[UNRESOLVED QUESTION]:** Cannot be resolved from the current local workspace without further access or explicit testing.
- **[MISSING SOURCE]:** The component exists as a compiled binary without source code in `/var/www/cortex`.

---

## 2. Status of Key Architectural Hypotheses

### 2.1 The `EmailApiProviderManagerService` Host Mapping Hypothesis

| Aspect | Status & Finding |
|---|---|
| **Preliminary Audit Hypothesis:** | Setting `$newInterface->ex_smtp_host = $provider->default_from_email;` was flagged as a suspected copy-paste defect that would break SMTP delivery. |
| **Deep-Audit Verification:** | **[VERIFIED FACT]** Traced across three separate services (`EmailApiProviderManagerService.php:35`, `GoogleCredentialServerSync.php:72`, `GoogleWorkspaceUserServerSync.php:62`). All three deliberately assign the sender email address to `ex_smtp_host`. |
| **Why It Does Not Break Relay Delivery:** | **[VERIFIED FACT]** The corresponding PowerMTA template (`delivery_server_pmta_smtp_relay_template`) routes to `127.0.0.1:2525`. It does not contain `[smtp_host]`. Delivery is handled by `cortex-relay` on port 2525. |
| **Where It Remains a Trap:** | **[VERIFIED FACT]** If an engineer attempts to use `ex_smtp_host` with a standard direct SMTP template (`delivery_server_pmta_smtp_template`), PowerMTA will attempt to resolve the email address as a domain, causing delivery failures. |

---

### 2.2 Microservices Source Code Availability

| Binary | Source Code Present? | Observed Runtime Contract | Missing Internal Evidence |
|---|---|---|---|
| **`cortex-master`** | **[MISSING SOURCE]** (Stripped 64-bit ELF Go binary) | Invocations via `-op=run-campaign`, `-op=run-pmta-accounting`, `-op=data-suppression`, `-op=install-server`. Reads `config.ini` and `/microservices/campaigns/{id}.json`. | Internal goroutine scheduling, in-memory suppression hash matching algorithms, and SMTP socket connection pooling. |
| **`cortex-relay`** | **[MISSING SOURCE]** (64-bit ELF Go binary in `/storage/small-server-setup/home-content/`) | Listens on port 2525. Internal strings confirm `*ses.Client`, `*google.cred`, `elastic`, `postmark`. Acts as SMTP-to-HTTP bridge. | Upstream HTTP retry logic, rate limit backoff, and exact mapping between SMTP headers and API credentials. |
| **`cortex-tracking`** | **[MISSING SOURCE]** (Stripped 64-bit ELF Go binary) | HTTP daemon listening on `0.0.0.0:8082`. Reads `config.ini` `[tracking]`, updates `cortex_data.email_campaign_engagements`. | Cryptographic decryption algorithm for URL tokens (`/rd/{token}`). |
| **`cortex-tools`** | **[MISSING SOURCE]** (Stripped 64-bit ELF Go binary) | Invocations via `-op=run-domain-blk-checker`, `-op=run-ip-blk-checker`. | List of hardcoded DNSBL servers queried. |
| **`networks_apis/*`** | **[MISSING SOURCE]** (Stripped 64-bit ELF Go binaries) | Invocations via `-op=get-suppression-link`, `-op=add-offers`, `-op=get-conversion-report`. | Internal HTTP client timeout and parsing rules for individual CPA network APIs. |

---

## 3. Unresolved Questions & Required Evidence

### Question 1: Delivery Server Relay Operational Status
- **Status:** **[UNRESOLVED QUESTION]**
- **Description:** Stored PowerMTA templates configure `route 127.0.0.1:2525 # smtp relay`. However, port 2525 is not listening on the master server host. It is designed to listen on remote delivery servers under `cortex-relay.service`.
- **Unknown:** Are active delivery servers currently running `cortex-relay` on port 2525, or are live campaigns running exclusively over direct IP Virtual MTAs (`delivery_server_pmta_vmta_template`)?
- **Evidence Needed to Resolve:** A read-only SSH check (`systemctl status cortex-relay` or `ss -tulpn | grep 2525`) on an active delivery server from `delivery_servers` table (requires explicit owner authorization).

---

### Question 2: Location of Go Microservice Repositories
- **Status:** **[UNRESOLVED QUESTION]**
- **Description:** The Go binaries in `/var/www/cortex/microservices/` were compiled from Go source code (Go Build IDs present in ELF headers), but the source repository is not in `/var/www/cortex`.
- **Unknown:** Is there an external Git repository hosting the Go microservices (`cortex-master`, `cortex-relay`, `cortex-tracking`)?
- **Evidence Needed to Resolve:** Clarification from the system owner regarding repository locations.

---

### Question 3: Upstream ESP Webhook Handling
- **Status:** **[VERIFIED FACT]**
- **Description:** For native HTTP email sending (e.g., Mailgun, SendGrid, SES), delivery events and bounces are delivered asynchronously via HTTP webhooks.
- **Audit Fact:** `routes/api.php` contains no webhook routes, and no controllers handle ESP webhook event payloads.
- **Evidence Needed for Extensions:** If a native HTTP provider is implemented, an inbound webhook controller and signature verification middleware must be designed and registered.

---

### Question 4: Backup Coverage Scope
- **Status:** **[VERIFIED FACT]**
- **Description:** `/var/www/cortex/environment/auto-backup.sh` dumps databases `cortex` and `cortex_data`.
- **Audit Fact:** The script does not back up filesystem configurations (`storage/app/settings.json`, `.env`, uploaded images).
- **Evidence Needed for Extensions:** Before any future staging clone is established, a full filesystem tarball must be taken alongside the database backup to ensure complete disaster recoverability.
