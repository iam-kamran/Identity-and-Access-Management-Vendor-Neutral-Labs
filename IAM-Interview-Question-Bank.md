**Author:** Kamran Arif  - https://www.linkedin.com/in/karifa/

# IAM Interview Prep Tracker

A running log of interview questions, with detailed answer options and optimal approaches. Updated after each interview.

---

## Q1: Explain the entire authentication flow when a user logs into an app

There isn't one universal flow — it depends on the protocol (SAML vs OIDC) and whether the app is SP-initiated or IdP-initiated. Give the interviewer the general shape first, then show you know the protocol-level detail.

### General shape (protocol-agnostic)

```
User (Browser)          Application (SP)              Identity Provider (IdP)
     |                        |                                |
     |--1. Access app-------->|                                |
     |                        |--2. No valid session,          |
     |                        |    redirect to IdP------------>|
     |<--3. Redirect (302)----|                                |
     |----4. Follow redirect to IdP login page---------------->|
     |<--5. Login page (username/password, or SSO cookie)------|
     |----6. Submit credentials-------------------------------->|
     |                        |     7. IdP validates creds,   |
     |                        |        checks MFA policy,      |
     |                        |        checks conditional      |
     |                        |        access / risk           |
     |<--8. Auth success, IdP issues token/assertion-----------|
     |----9. POST token/assertion to app (ACS URL / redirect_uri)->|
     |                        |--10. App validates signature, |
     |                        |     issuer, audience, expiry   |
     |<--11. App creates session, sets cookie, grants access---|
```

### Step-by-step detail

1. **User requests a protected resource.** App checks for a valid local session (cookie). If none, it triggers auth.
2. **SP-initiated redirect.** App redirects browser to the IdP with a request:
   - SAML: `AuthnRequest` (XML, base64-encoded, sent via redirect or POST binding)
   - OIDC: `/authorize` request with `client_id`, `redirect_uri`, `scope`, `response_type`, `state`, `nonce`
3. **IdP authenticates the user.** If no existing IdP session, prompts for credentials. If existing session (SSO cookie) exists, this step is skipped — this is what "SSO" actually delivers.
4. **Policy evaluation.** IdP evaluates MFA policy, conditional access / adaptive risk (device compliance, location, IP reputation, impossible travel), and group/app assignment.
5. **Token/assertion issuance.**
   - SAML: IdP builds a signed XML `Assertion` containing `NameID`, attributes, `AudienceRestriction`, `Conditions` (NotBefore/NotOnOrAfter), and posts it to the app's ACS URL.
   - OIDC: IdP returns an **authorization code** to the redirect_uri (front channel). App's backend then exchanges the code for an **ID token** (JWT — proves identity) and **access token** (JWT/opaque — used to call APIs) via the token endpoint (back channel, using client secret/certificate).
6. **App validates the token.**
   - Signature (against IdP's public key / metadata)
   - Issuer (`iss`) matches expected IdP
   - Audience (`aud`) matches this app's client ID
   - Expiry (`exp`) and not-before (`nbf`)
   - Nonce/state match what was sent (CSRF/replay protection)
7. **Session establishment.** App creates its own local session (cookie, usually short-lived + refresh token in OIDC), and the user is now logged in.

### Key answer points interviewers look for
- Difference between **authentication** (who you are — IdP's job) and **authorization** (what you can do — app's job, sometimes federated too via scopes/roles in the token).
- SAML vs OIDC: SAML is XML/assertion-based, common in legacy enterprise SSO; OIDC is JSON/JWT-based, built on OAuth2, more common in modern/mobile/API-first apps.
- Where MFA and Conditional Access actually get enforced — at the IdP, before the token is issued, not at the app.
- SSO mechanism = the IdP session cookie; that's what avoids re-prompting across apps.

### Reference
- [Microsoft: SAML and OAuth/OIDC protocols overview](https://learn.microsoft.com/en-us/entra/identity-platform/authentication-vs-authorization)
- [Okta Developer: Authentication flows](https://developer.okta.com/docs/concepts/auth-overview/)

---

## Q2: Company uses Okta as IdP, acquires an org using Entra ID — how do you onboard those users onto Okta?

This is an M&A identity integration question. Interviewers want to see you think in **phases**, not a single big-bang cutover. Frame it as Discovery → Coexistence → Migration → Decommission.

### Phase 1: Discovery
- Inventory the acquired org's Entra tenant: user count, groups, licensed apps (SaaS apps using Entra as SSO), on-prem AD (if hybrid), MFA methods in use, conditional access policies.
- Identify overlapping domains/UPN conflicts (very common in M&A — same email domain or duplicate identities).
- Classify apps by criticality/complexity for migration sequencing.

### Phase 2: Coexistence (quick win — minimize immediate disruption)
Two common patterns, pick based on urgency:

**Option A — Okta as the hub, Entra federated in temporarily**
- Configure Entra ID as an **Identity Provider (IdP) in Okta** (Okta supports Entra/Azure AD as a SAML/OIDC IdP via Okta's "Identity Providers" + routing rules).
- Set up an Okta **routing rule** (IdP Discovery) so acquired-org users authenticate against Entra initially, while getting access to shared/Okta-fronted apps via a JIT (Just-In-Time) provisioned Okta profile.
- This gives immediate access to shared resources (email, Slack, shared SaaS apps) without forcing a password reset day one.

**Option B — Direct app re-point (if few shared apps)**
- Re-point the specific SaaS apps the acquired org needs access to, directly from Entra SSO to Okta SSO, app by app, without a full federation trust.

### Phase 3: Identity Migration (the real work)
- **Directory sync**: Bring acquired org's identities into Okta Universal Directory.
  - If they have on-prem AD → deploy Okta AD Agent, import via LDAP/AD import.
  - If cloud-only Entra ID → use Okta's Entra ID (Azure AD) integration/importer, or SCIM-based sync, or a scripted export/import (Graph API → Okta Users API) for a one-time bulk migration with an ImmutableId-style correlation attribute to prevent duplicate accounts.
- **Identity correlation**: match by email/UPN, flag conflicts for manual resolution before import (same problem class as SailPoint/Entra soft-match — you've hit this in your lab work too).
- **MFA re-enrollment**: users enroll in Okta Verify / your org's MFA standard; can't migrate MFA factors across platforms.
- **Group/app assignment**: rebuild group-based app assignment logic in Okta (Entra security groups → Okta groups, either by group push/sync mapping or manual rebuild if org structure is changing).
- **Password**: either password import (if compliant hash format available) or force reset on first Okta login — most orgs choose forced reset for security cleanliness post-M&A.
- **Communication + pilot**: migrate in waves (pilot group → department → org-wide), not all at once.

### Phase 4: Cutover & Decommission
- Once all target users/apps are validated on Okta, disable the Entra federation trust / routing rule.
- Deprovision the legacy Entra tenant's app registrations for migrated apps.
- Decide fate of the Entra tenant itself — decommission, or retain for other acquired-org workloads (e.g., if they keep an Azure subscription tied to that tenant).

### What makes this answer "optimal" in an interview
- Show you understand **why phased beats big-bang**: business continuity, minimizing helpdesk load, and reducing user disruption during a merger (already a sensitive time).
- Mention **duplicate identity/UPN conflict handling** — this is the detail that separates a senior answer from a junior one.
- Mention **SOD/access review implications** — post-migration, entitlements should go through a governance review (tie-in to your SailPoint/IGA background) before final decommission.

### Reference
- [Okta: Identity Providers (federate with Azure AD/Entra)](https://help.okta.com/en-us/content/topics/security/idp-add-generic.htm)
- [Okta: Directory integrations](https://www.okta.com/integrations/microsoft-azure-active-directory/)

---

## Q3: Your role as Cloud SME when JML is configured by Saviynt IGA personnel

*(Interview question paraphrased: "You mentioned you were part of JML configuration in a cloud environment via Saviynt — what was your role as Cloud SME when the actual JML workflows were configured by the Saviynt IGA team?")*

The key here is drawing a clean line between **what the IGA/Saviynt team owns** vs **what the Cloud SME owns** — interviewers ask this specifically to check you're not overclaiming platform-config work that wasn't yours.

### Division of responsibilities

| Area | Saviynt IGA Team | Cloud SME (your role) |
|---|---|---|
| JML workflow logic (approval chains, birthright rules, SOD policy engine) | Owns configuration | Provides business/technical requirements input |
| Cloud connector setup (Azure/AWS/GCP connector to Saviynt) | Configures connector in Saviynt | Provides target environment details — subscription/tenant structure, RBAC role definitions, resource hierarchy |
| Entitlement catalog for cloud resources | Imports/maps entitlements into Saviynt | **Defines and validates** what entitlements/roles exist in the cloud environment (e.g., Azure custom RBAC roles, AWS IAM policies) so they're accurately represented |
| Birthright access rules for cloud roles | Builds the rule logic | Advises which cloud roles are appropriate as birthright vs request-based, based on least-privilege |
| Provisioning/deprovisioning execution failures on cloud side | Triggers the JML event | **Troubleshoots** — e.g., a Leaver event fires but the Azure role assignment doesn't get removed due to a scoping or permission issue on the service principal Saviynt uses |
| Access certification / recertification campaigns | Runs the campaign in Saviynt | Validates that cloud entitlement data being certified is accurate and current |
| SOD conflict rules involving cloud permissions | Defines/configures the SOD rule set | Provides subject-matter input on which cloud role combinations are actually toxic (e.g., a role that can both create and approve resource deployments) |

### A concrete way to phrase it in an interview
"The IGA team owned the Saviynt platform itself — workflow design, approval hierarchies, the SOD rule engine. My role as Cloud SME was to be the bridge between the cloud environment's actual permission model and what Saviynt represents. That meant validating the entitlement catalog matched real Azure/AWS RBAC roles, troubleshooting when the Saviynt connector's service principal didn't have the right scope to execute a deprovisioning action, and advising on which roles should be birthright versus request-only based on least-privilege principles. I wasn't configuring the JML workflow logic — I was making sure the cloud side of that workflow was accurate and functioning."

### Follow-up they may ask
- "Give an example of a provisioning failure you troubleshot." — have one concrete example ready (e.g., a scoping issue, an expired service principal secret, a role not existing at the expected scope).
- "How did you validate the entitlement catalog was accurate?" — mention comparing Saviynt's imported entitlements against a live `az role definition list` / IAM policy export, catching drift.

---

## Q4: Have you integrated Sentinel, Purview, Intune in your tenant? / What's your exposure to Agent ID, agentic AI identity workloads, and authentication for AI Foundry / Vertex AI?

This was a multi-part question probing breadth across the Microsoft security stack and current agentic-AI identity concepts. Break it into parts — don't try to give one blended answer.

### Part A — Sentinel, Purview, Intune integration with Entra ID

These three are the standard "Microsoft security stack" companions to Entra, and IAM interviewers ask this to see if you understand how identity plugs into the broader security ecosystem, not just Entra in isolation. Answer honestly based on your actual hands-on exposure — use this as the framework to describe what you *have* touched and be direct about what you haven't:

| Product | What the integration with Entra actually looks like |
|---|---|
| **Microsoft Sentinel** (SIEM/SOAR) | Ingests Entra ID sign-in logs, audit logs, and Identity Protection risk events as a data connector. IAM-relevant use: building analytics rules/detections on identity signals — e.g., impossible travel, risky sign-ins, privileged role activations outside business hours — and feeding those into incident response playbooks (Logic Apps) that can auto-disable a user or revoke sessions via Graph API. |
| **Microsoft Purview** (data governance/compliance) | Uses Entra ID for identity context on data access — who accessed what sensitive data, sensitivity labels tied to user/group identity, DLP policies that key off Entra group membership. IAM-relevant use: access reviews and entitlement management can incorporate Purview's data classification to prioritize which access grants are highest-risk (e.g., access to data labeled "highly confidential"). |
| **Microsoft Intune** (device management/MDM) | Feeds device compliance state into Entra Conditional Access — "require compliant device" is an Intune-sourced signal enforced at the Entra policy layer. IAM-relevant use: this is where identity and device posture intersect — a user can have correct entitlements but still be blocked if their device isn't Intune-compliant/enrolled. |

**If your hands-on exposure is limited**, the honest and still-strong answer is: "My direct hands-on work has been on the Entra/IAM side — provisioning, SSO, governance. I understand how Sentinel, Purview, and Intune consume Entra's identity signals and Conditional Access to extend protection, but I haven't personally configured those platforms." That's a legitimate answer — don't overclaim tools you haven't touched, since a follow-up drill-down question will expose it fast.

### Part B — Microsoft Entra Agent ID / identity workloads for agentic AI

This is a genuinely new area (rolled out through 2026) and a strong area to speak to given your background — it's IAM applied to a new class of non-human identity.

**Why it exists:** AI agents (assistive or autonomous) need to authenticate and be governed, but they don't fit the two existing identity models — they're not human users (no passwords/MFA) and not static service principals (they're created/destroyed dynamically, sometimes thousands of times a day).

**Core constructs:**
- **Agent identity blueprint** — a reusable template (like an app registration) that defines an agent's type, permissions, and metadata. Published once per agent product/pattern.
- **Agent identity** — the actual account instantiated from a blueprint, one per deployed agent instance. Has an object ID/app ID, no credentials of its own.
- **Agent's user account (optional)** — a paired Entra user account for agents that must access systems requiring a true user object (e.g., a mailbox or Teams channel).

**Authentication mechanism:** Agent identities never hold credentials directly. They authenticate using **federated identity credentials (FIC)** issued by the blueprint — the blueprint holds the actual credential (client secret, certificate, or, best practice, a **federated trust to a managed identity**), and agents get short-lived tokens via OAuth 2.0/OIDC client-credentials-style flows. This mirrors the "no static secrets" pattern IAM has been pushing for workload identities generally.

**Governance angle (this is your strongest ground):** Agent identities plug into the same Entra ID Governance capabilities as human identities — lifecycle management, access reviews, entitlement management via access packages, and a mandatory **sponsor** (a named human accountable for the agent). Conditional Access and Identity Protection also extend to agents — risk-based policies can block a high-risk agent the same way they'd block a risky user sign-in.

**How to frame this in an interview:** "Agent ID essentially extends the same governance principles I already apply to human joiner-mover-leaver processes — least privilege, named accountability, lifecycle expiration — to a new identity type that's more dynamic and higher-volume than either users or traditional service principals."

### Part C — Authentication for Azure AI Foundry

Foundry supports two methods:

1. **API keys** — static secret in the `Ocp-Apim-Subscription-Key` header. Fine for prototyping, but no per-user auditing and a static-secret risk — the kind of thing IAM would flag in a review.
2. **Microsoft Entra ID (recommended for production)** — OAuth 2.0 bearer tokens scoped to the resource's audience (e.g., `https://ai.azure.com/.default`), issued to a **user, service principal, or managed identity**, and authorized via **Azure RBAC** (e.g., "Cognitive Services User" role scoped to the Foundry resource). Best practice is to disable key-based auth entirely once Entra ID is configured (`-DisableLocalAuth $true`), and to use a **managed identity** so no secret ever needs to be stored or rotated manually.

For **agents built inside Foundry** specifically, authentication chains through Agent ID: the agent identity blueprint has a federated credential trust with the Foundry project's managed identity → the managed identity authenticates the blueprint to Entra ID (no secret) → Entra issues a token for the agent identity → that token is exchanged for a scoped access token to call the downstream resource (e.g., Azure Storage). This is the same "no static secrets" chain as Part B, applied specifically to Foundry's agent runtime.

### Part D — Authentication for Google Vertex AI (now part of Gemini Enterprise Agent Platform)

Google's model is IAM-centric rather than Entra-style app registrations:

1. **Application Default Credentials (ADC)** — the client libraries auto-discover credentials in a defined order: local user credentials (via `gcloud auth login`) for dev, or the **service account attached to the compute resource** (Compute Engine, GKE, Cloud Run) in production.
2. **Service accounts + IAM roles** — the standard production pattern: attach a service account to the compute resource, grant it a role like `roles/aiplatform.user`, and workloads inherit that identity automatically — conceptually equivalent to an Azure managed identity plus RBAC role assignment.
3. **Workload Identity Federation (WIF)** — Google's preferred method for **cross-cloud** access (e.g., calling Vertex AI from AWS or Azure, or from an on-prem/non-Google IdP). An external token (AWS STS token, Azure AD token, OIDC token from a CI/CD system) is exchanged via Google's Security Token Service for a short-lived federated Google token, which can then directly access resources or impersonate a service account — no long-lived Google service account key ever needs to leave GCP. This is the direct Google equivalent of Entra's federated identity credentials (FIC) pattern.
4. **Google-managed agent identities** — for agentic workloads specifically (Vertex AI Agent Engine), Google now offers attested, lifecycle-bound **agent identities** as a more secure alternative to service accounts, governed through the same IAM role model.

**Cross-cloud framing that lands well in an interview:** "Both Microsoft and Google have converged on the same principle for workload/agent identity — eliminate long-lived static secrets in favor of short-lived, federated tokens issued through a trust relationship (FIC in Entra, Workload Identity Federation in GCP). The mechanics differ, but the IAM pattern — least privilege, no standing secrets, centrally governed trust — is identical."

### Reference
- [Microsoft Entra Agent ID overview](https://learn.microsoft.com/en-us/entra/agent-id/what-is-microsoft-entra-agent-id)
- [What are agent identities?](https://learn.microsoft.com/en-us/entra/agent-id/what-are-agent-identities)
- [Foundry authentication and authorization](https://learn.microsoft.com/en-us/azure/foundry/concepts/authentication-authorization-foundry)
- [Foundry agent identity concepts](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)
- [Google Cloud: Authenticate to Vertex AI](https://cloud.google.com/vertex-ai/docs/authentication)
- [Google Cloud: Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation)

---

## Q5: What are common SSO issues you get as an IAM SSO Ops person, and how do you resolve them?

This question tests operational/BAU (business-as-usual) troubleshooting depth, not architecture knowledge. Interviewers want to hear that you can triage fast, know exactly where in the flow to look, and have a repeatable method — not just "I checked the logs." Structure your answer by **where in the auth flow the failure occurs**, since that's how you'd actually triage it live.

### 1. Certificate / signing key expiry (the most common one)
**Symptom:** SSO that worked yesterday suddenly fails for *everyone* on one app — usually with a vague "invalid signature" or "authentication failed" error at the SP.
**Cause:** SAML signing certificate on the IdP (or the SP's verification cert on the IdP side) expired, or was rotated without updating the other side's metadata.
**Resolution:** Check cert expiry first in any outage triage (`openssl x509 -enddate` or the IdP admin console). Rotate/renew and re-upload the new metadata to the SP (or vice versa). **Prevention** is the senior-level answer here: set up expiry alerting 30-60 days out, and prefer IdPs/SPs that support **metadata auto-refresh via a federation metadata URL** instead of manually pasted certs, so rotation doesn't require a coordinated change on both sides.

### 2. Clock skew
**Symptom:** Intermittent failures, often "assertion not yet valid" or "assertion expired" errors, especially right after a server reboot or on VMs.
**Cause:** SAML assertions carry `NotBefore`/`NotOnOrAfter` timestamps with a tight validity window (often just a few minutes); if the SP or IdP server clock drifts, valid assertions get rejected.
<br>**Resolution:** Verify NTP sync on both sides. Most IdPs allow a small clock-skew tolerance setting — check it's configured (typically 3-5 minutes) rather than assuming default.

### 3. Attribute/claim mapping mismatch
**Symptom:** User authenticates successfully at the IdP (you can see a successful sign-in in the IdP logs) but the app either errors on login or logs them in as the wrong user/wrong permissions.
**Cause:** The SP expects a specific attribute (e.g., `NameID` = email, or a custom claim like `employeeID`) but the IdP is sending a different format, or a required attribute isn't being released at all.
**Resolution:** This is a classic "logs on both sides" problem — the IdP will show success, so you have to pull the actual SAML assertion (browser dev tools / SAML-tracer extension, or the SP's rejected-request log) to see what attributes were actually sent, and diff that against the SP's documented attribute contract. Fix by correcting the attribute mapping/claims rule on the IdP side.

### 4. Audience/Entity ID (Issuer) mismatch
**Symptom:** Hard failure, usually clear: "audience restriction" or "issuer not recognized" error.
**Cause:** SP was reconfigured (new environment, new URL after a domain change) but the IdP's app configuration still points to the old Entity ID/ACS URL, or vice versa — very common after an app migrates environments (dev→prod, or a re-platform).
**Resolution:** Compare the SP's metadata (Entity ID, ACS URL, audience) against exactly what's configured on the IdP app registration — these have to match precisely.

### 5. Redirect loop / "keeps sending me back to login"
**Symptom:** User authenticates, gets redirected back to the app, and immediately gets redirected to log in again — repeats indefinitely.
**Cause:** Almost always a **session/cookie problem**, not an auth failure — third-party cookies being blocked by the browser, SameSite cookie policy misconfiguration, or the app failing to actually persist the session after receiving a valid assertion/token.
**Resolution:** Confirm the assertion/token IS being accepted (check IdP logs show success) — if so, this is an app-side session bug, not an identity problem; loop in the app team. If it's cookie-policy related, check whether the app relies on third-party cookies and browser settings/extensions are blocking them.

### 6. JIT (Just-In-Time) provisioning failures
**Symptom:** New user gets a valid SSO login but lands on an error page or gets a "user not found" response from the app.
**Cause:** The app relies on JIT provisioning driven by SAML/SCIM attributes, but a required attribute for account creation (e.g., a unique employee ID) is missing or malformed in the assertion, so the app can't create the local user record.
**Resolution:** Same diagnostic approach as #3 — inspect the actual assertion payload against what the app's JIT logic requires. Often surfaces during onboarding waves for a new app, not steady-state.

### 7. MFA/Conditional Access blocking silently
**Symptom:** User reports "SSO isn't working" but it's actually an access-denial, not a technical failure — they get blocked or endlessly prompted for MFA.
**Cause:** A Conditional Access policy (device compliance, location, risk-based) is correctly doing its job and blocking the sign-in — but from the user's perspective it looks like "SSO is broken."
**Resolution:** Check Identity Protection / Conditional Access sign-in logs (Entra) first — the "failure reason" field usually states the exact policy that blocked it. This is why triage order matters: rule out policy-based *intentional* denial before treating it as a bug.

### 8. Stale group/role sync causing "logged in but wrong access"
**Symptom:** SSO succeeds, but the user has the wrong app-level permissions (too much or too little access).
**Cause:** App relies on group claims in the token/assertion to map to app roles, but the user's group membership changed in the IdP and hasn't synced/propagated, or the app is caching stale role data.
**Resolution:** Distinguish IdP-side (has the token/assertion actually updated? force a fresh sign-in) from app-side (is the app caching role mappings and needs a cache clear/re-sync) — this determines who owns the fix.

### General SSO Ops triage method (say this explicitly — it shows maturity)
1. **Isolate scope first** — one user, one app, or everyone? This alone rules out half the causes (cert/audience issues are systemic; attribute/JIT issues are usually per-user or per-new-user).
2. **Check the IdP logs first** — did authentication succeed at the IdP? This splits the problem into "IdP-side" vs "SP-side" immediately.
3. **Pull the actual assertion/token** when logs alone aren't conclusive — this is the single most useful troubleshooting step and separates senior SSO Ops people from junior ones.
4. **Check cert/clock/audience basics** before deep-diving — they're the most common root cause and the fastest to rule out.

### Reference
- [Microsoft: Troubleshoot SAML-based single sign-on](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/debug-saml-sso-issues)
- [Okta: Troubleshooting SSO issues](https://help.okta.com/en-us/content/topics/apps/apps_sso_troubleshooting.htm)

---

## Q6: If you got a fresh tenant, what Conditional Access policies would you add? (Asked at IBM)

This tests whether you have a repeatable baseline methodology, not a random list. Structure the answer as: **prerequisites first, then the baseline policy set, then maturity add-ons** — that ordering itself signals experience, because deploying CA without the prerequisites is a classic way to lock yourself out of the tenant.

### Prerequisites — before writing a single policy
1. **Create break-glass (emergency access) accounts** — 2 cloud-only accounts, excluded from every CA policy, with long random passwords stored securely (not in the same vault a policy could lock you out of) and no MFA registered. This is the single most important step and interviewers listen for it specifically — CA misconfiguration is a top cause of full tenant lockout.
2. **Deploy in "Report-only" mode first** — every policy should run in report-only and be checked against the Sign-in logs / Conditional Access Insights workbook before switching to "On," to catch unintended blocks (e.g., a service account that doesn't support MFA).
3. **Baseline the current traffic** — export sign-in logs to see per-app MFA usage, legacy auth volume, and named locations in use, rather than authoring policies on assumptions.

### The baseline policy set (minimum for any tenant)

| # | Policy | Scope | Grant control |
|---|---|---|---|
| 1 | Require MFA for admins | All Entra privileged roles | MFA, ideally phishing-resistant (FIDO2/Windows Hello) |
| 2 | Require MFA for all users | All users, excluding break-glass | MFA |
| 3 | Block legacy authentication | All users, legacy auth clients (POP/IMAP/SMTP/older Office clients) | Block |
| 4 | Require compliant or hybrid-joined device for M365 access | All users, M365 apps | Grant: compliant device OR hybrid Entra joined |
| 5 | Sign-in risk-based policy (High risk) | All users | Require MFA — needs Entra ID P2 |
| 6 | User risk-based policy (High risk) | All users | Require secure password change — needs Entra ID P2 |

If the org is on P1 only (no P2), swap 5/6 for **Authentication strength / sign-in frequency policies** as a partial substitute, and say so — shows you understand licensing constraints shape design, not just theory.

### Maturity add-ons (mention these to show depth beyond the baseline six)
- **Block or restrict access from unfamiliar/high-risk countries** using named locations, if the business has a defined geographic footprint.
- **Require compliant device for admin portal access** (Azure portal, Entra admin center) specifically — separate from the general M365 policy, since admin surface is higher-risk.
- **Session controls** — sign-in frequency and "persistent browser session" settings for unmanaged/BYOD devices, rather than a blanket block, to balance security with usability on personal devices.
- **Terms of Use / acceptable use policy enforcement** at first sign-in, for compliance/legal coverage.
- **Guest/external user restrictions** — tighter Conditional Access for B2B guest accounts (e.g., mandatory MFA regardless of home tenant's policy).
- **Block unsupported/legacy device platforms** if the org has a defined supported-OS list.
- **PIM integration** — require CA-enforced MFA as a prerequisite for activating privileged roles via Privileged Identity Management, not just for the standing sign-in.

### The point to make explicitly in the interview
"I wouldn't just list policies — the sequence matters. Break-glass accounts first, everything staged in report-only before enforcement, and the baseline six as the floor, not the ceiling. Then I'd layer maturity policies based on the org's actual risk profile and licensing tier, using sign-in log data rather than guessing at what traffic looks like."

### Reference
- [Microsoft: Plan a Conditional Access deployment](https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access)
- [Microsoft: Emergency access (break-glass) accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)
- [Microsoft: Conditional Access common policies](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-policy-common)

---

## Q7: If your Super Admin (Global Admin) account is compromised, what are your action items as an IAM expert?

This is an incident response question. Interviewers want a **sequenced** answer — contain first, investigate second, recover third, prevent recurrence last — not a jumbled list. Saying the phases out loud before diving into detail is itself a strong signal.

### Phase 1: Immediate containment (minutes, not hours)
1. **Use a break-glass account to regain control** — sign in with your emergency access account (excluded from Conditional Access, MFA pre-registered) so you're not locked out while responding. This is why break-glass accounts exist in the first place — this scenario is exactly what they're for.
2. **Revoke all active sessions/refresh tokens for the compromised account** — `Revoke-MgUserSignInSession` / Entra admin center "Revoke sessions." A password reset alone is not enough — existing tokens remain valid until explicitly revoked.
3. **Force password reset** on the compromised account, and **disable the account** entirely if it's not needed for immediate recovery actions.
4. **Remove/reset MFA methods** registered on the account — attackers often register their own MFA method (e.g., their own phone number or authenticator) after compromise to maintain persistent access even after a password reset.
5. **Check for and remove any backdoors the attacker may have planted while they had Global Admin**, since GA access means they could have created new persistence mechanisms:
   - New Global Admin or privileged role assignments (check Entra audit logs for role assignment events)
   - New app registrations / service principals with high-privilege API permissions or client secrets they control
   - New federation/trust configurations (a rogue domain federation is a classic GA-level attack — redirects auth for a whole domain to an attacker-controlled IdP)
   - New conditional access policy exclusions (attacker weakening CA to make re-entry easier)
   - New mail forwarding rules or mailbox delegation (if GA had mailbox access via admin consent)
   - New guest users / B2B invitations

### Phase 2: Investigation (determine scope and root cause)
6. **Pull sign-in and audit logs** for the account's full compromised window — identify source IPs, locations, user agents, and every action taken while compromised. Entra sign-in logs + audit logs + Microsoft Sentinel/Defender for Identity if available.
7. **Determine the initial access vector** — phishing, token theft (AiTM/adversary-in-the-middle), credential stuffing, leaked credential, or an insider. This determines what else needs remediation (e.g., if it was a phished token, check if MFA was even bypassed via token replay — meaning MFA re-enrollment alone doesn't fully remediate).
8. **Check for lateral movement** — did the attacker use GA access to pivot into other systems (Azure resource access via a Global Admin's elevated access toggle, other privileged accounts, downstream SaaS apps via SSO)?

### Phase 3: Recovery
9. **Rotate credentials/secrets the compromised GA could have accessed or exposed** — any certificates, client secrets, or Key Vault access the account had rights to.
10. **Re-validate every privileged role assignment** in the tenant against expected state — treat this as a point-in-time access review of all admin roles, not just the compromised account.
11. **Restore any legitimately-needed configuration** the attacker tampered with (CA policies, federation settings) from a known-good baseline.

### Phase 4: Prevention / hardening (what you'd push for afterward)
12. **Move Global Admin to PIM-eligible, not standing, access** — if the account had standing GA rather than just-in-time activation via Privileged Identity Management, that's the root architectural gap to fix.
13. **Enforce phishing-resistant MFA** (FIDO2 security keys, certificate-based auth) specifically for all privileged roles — if the compromise happened via a phishable MFA method (SMS/push), this is the direct fix.
14. **Reduce standing Global Admin count** — Microsoft's own guidance is fewer than 5 permanent GAs; most organizations should be using scoped, least-privilege roles instead of GA for day-to-day admin tasks.
15. **Enable Conditional Access specifically for admin roles** requiring compliant device + phishing-resistant MFA + trusted location, separate from the general-user policy.
16. **Post-incident review** — document timeline, root cause, and remediation for compliance/audit purposes (this ties back to governance — expect an access review and possibly a SOC/compliance notification requirement depending on the industry).

### The framing that shows seniority
"The instinct is to jump straight to 'reset the password,' but a password reset doesn't revoke existing tokens and doesn't address anything the attacker planted while they had admin rights. The real work is: contain with break-glass access, revoke sessions, then audit everything a Global Admin *could* have touched — new admins, new app registrations, federation changes, CA exclusions — because GA is effectively full tenant control, not just one account's access."

### Reference
- [Microsoft: Respond to a compromised Global Administrator](https://learn.microsoft.com/en-us/entra/identity/secondary-authentication-factor-lost-or-stolen)
- [Microsoft: Secure privileged access](https://learn.microsoft.com/en-us/security/privileged-access-workstations/overview)
- [Microsoft: Emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)

