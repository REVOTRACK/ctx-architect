# CORTEX_CHANGE_PLAYBOOK.md
**Standard Operating Procedure for Safe Extensions, Integrations & Changes**  
**Audit Date:** 2026-10-11  
**Operating Mode:** Mandatory Engineering Governance

---

## 1. The Core Operating Mandate

Every future engineer or AI agent working on Cortex must adhere to this principle:
> **Extend the existing system; do not redesign, replace, or build parallel infrastructure.**
> Preservation of working production functionality takes absolute precedence over speed, refactoring, or modernizing technologies.

---

## 2. Mandatory Approval Gates

Before any production file or server setting is modified, the change must pass through seven formal gates in sequence:

```
[ Gate A: Plan & Discovery ]
             │
             ▼
[ Gate B: Backup Verification (Static) ]
             │
             ▼
[ Gate C: Backup & Isolated Clone Creation ]
             │
             ▼
[ Gate D: Implementation in Clone Only ]
             │
             ▼
[ Gate E: Offline & Sandbox Testing ]
             │
             ▼
[ Gate F: Diff & Risk Review ]
             │
             ▼
[ Gate G: Explicit Deployment Approval ]
```

### Gate A: Plan & Discovery
- Define the required capability precisely.
- Search Cortex for the closest existing implementation.
- Trace that implementation end to end across UI, controller, model, service, configuration, microservice, and external systems.
- Present the plan and stop. No file changes permitted.

### Gate B: Backup Verification (Static Review)
- Statically review `/var/www/cortex/environment/auto-backup.sh` and `/var/www/cortex/environment/auto-restore.sh`.
- Validate that the backup script has a working test mode (`-t/--test`).
- Identify any gaps in automated backup coverage (e.g., filesystem assets, settings).
- Obtain explicit approval before executing backup commands.

### Gate C: Backup & Isolated Clone Creation
- Execute backup verification.
- Create a timestamped, verifiable archive of the application code, configurations, and MySQL schemas (`cortex`, `cortex_data`).
- Clone the verified codebase into a strictly isolated staging directory (e.g., `/var/www/cortex-staging`).
- **Prove Isolation:**
  - The clone must connect to an isolated, non-production test database.
  - The clone must point outbound SMTP/MTA configurations to a local test sink (e.g., Mailpit / MailHog).
  - The clone must not share live production API credentials or PowerMTA listeners.
  - A simple directory copy without configuration isolation does not qualify as an isolated clone.

### Gate D: Implementation in Clone Only
- Perform all coding exclusively within the isolated clone directory.
- Never edit files in `/var/www/cortex` during development.
- Keep diffs small, focused, and minimal. Do not refactor unrelated code.

### Gate E: Offline & Sandbox Testing
- Execute tests exclusively using mocks, internal fixtures, and provider sandboxes.
- Never send test emails to real recipients or unwhitelisted addresses.
- Never trigger live affiliate campaigns or billable cloud APIs.
- Validate that suppression, audit checks, and tracking mechanisms function without regression.

### Gate F: Diff & Risk Review
- Generate a unified diff (`git diff` or `diff -uprN`) between the production directory and the clone.
- Review every modified line against existing Cortex conventions.
- Document any observed limitations, edge cases, and failure modes.
- Prepare a tested, one-step rollback procedure.

### Gate G: Explicit Deployment Approval
- Submit the diff, test evidence, and rollback procedure to the system owner.
- Wait for explicit written approval before scheduling deployment.

---

## 3. Provider Extension Evaluation Matrix

Before proposing an email or custom API provider integration, complete this evaluation:

| Dimension | Option A: PowerMTA SMTP Relay Pattern (Preferred) | Option B: Native HTTP REST API Pattern |
|---|---|---|
| **Architecture Fit** | **100% Consistent.** Follows existing `DeliveryServerInterface` and `delivery_server_pmta_smtp_template`. | **Deviates from Core Engine.** Requires building parallel sending workers. |
| **Throughput & Speed** | High. PowerMTA manages connection pooling, pipelining, and socket reuse. | Moderate. Constrained by PHP process memory and HTTP latency per message. |
| **Engine Compatibility** | `cortex-master` injects via local SMTP seamlessly. Zero binary changes needed. | `cortex-master` cannot call HTTP email APIs; must be bypassed. |
| **Tracking Attribution** | `cortex-tracking` works automatically via pre-rendered campaign tokens. | Requires custom webhook handlers to translate provider events back to `cortex_data`. |
| **Accounting & Hygiene** | `cortex-master -op=run-pmta-accounting` parses bounces automatically. | Requires inbound webhook receivers for bounce/complaint webhooks in `routes/api.php`. |
| **Recommended Decision** | **Use Option A whenever provider supports SMTP relay.** | **Use Option B only if provider is strictly HTTP-only.** |

---

## 4. Reusable Feature Checklist & Decision Template

Every future proposed feature or provider extension must conclude with this completed template:

```markdown
### Feature Decision Template: [Feature / Provider Name]

1. **Requested Behavior:**
   [State the precise capability required, e.g., outbound delivery via SendGrid, inbound webhook sync, etc.]

2. **Closest Existing Cortex Pattern:**
   [Identify the exact reference class/pattern in Cortex, e.g., CloudFlareApiService for REST, or INTERFACE_TYPE_SMTP_THROUGH_PMTA for SMTP.]

3. **Verified Source References:**
   - Model: [e.g., app/Models/DeliveryServerInterface.php:29-50]
   - Controller / UI: [e.g., app/Livewire/EmailApiProviderManager.php:140-160]
   - Service: [e.g., app/Services/DeliveryServer/PowerMtaConfigGenerator.php:42-49]
   - Template: [e.g., storage/app/settings.json -> delivery_server_pmta_smtp_template]

4. **Can Existing Architecture Support It?**
   [Yes / Yes with Prerequisites / No / Unclear - cite exact code evidence]

5. **Smallest Architecture-Consistent Approach:**
   [Describe the minimal diff required to integrate using established conventions]

6. **New Components Required (if any) and Why Unavoidable:**
   [Justify every new file, class, or database field]

7. **Exact Files That Would Change:**
   - File 1: [Path and proposed modification]
   - File 2: [Path and proposed modification]

8. **Potential Impact on Existing Campaigns / Providers / Data:**
   [Analyze regression risks for running campaigns and existing MTA interfaces]

9. **Security, Rate Limiting & Operational Risks:**
   [Credential storage, credential transmission, quota enforcement, IP limits]

10. **Isolation & Test Plan:**
    - Isolation proof: [How test environment is isolated from live DB and live SMTP]
    - Unit tests: [Contract tests for payloads and templates]
    - Sandbox tests: [Mock SMTP or provider sandbox verification]

11. **Rollback Plan:**
    [Exact command or reverse patch to restore previous state immediately]

12. **Approval Status:**
    [Explicit Approval Required. No implementation permitted until authorized.]
```
