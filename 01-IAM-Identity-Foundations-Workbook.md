# Identity Foundation Notebook — Theory + Architecture Workbook

**Author:** Kamran Arif  - https://www.linkedin.com/in/karifa/

> **Purpose:** A vendor-neutral identity foundation for daily study.
>
> **Source foundation:** This notebook preserves and reorganizes the theory, terminology, explanations, and security concepts from the uploaded *Identity Foundations Notes* reference. It then adds clearly separated architecture workflows and study questions.
>
> **Daily reading target:** 20–30 minutes.

---

## How to Read This Notebook

Each chapter has four layers:

1. **Theory** — the underlying identity concept and terminology.
2. **Mental model** — the simplest way to remember it.
3. **Architecture workflow** — how the concept moves through an enterprise system.
4. **Architect's questions** — how to apply it in a real design.

The goal is not to memorize product screens. Learn the identity model first; map products onto it later.

---

# Part I — Identity Security Foundations

## 1. Authentication vs Authorization

### Authentication (AuthN)

Authentication confirms that a human or non-human entity is who they claim to be by validating evidence such as passwords, biometrics, certificates, or other authenticators.

**Question answered:**

> Who are you?

### Authorization (AuthZ)

Authorization determines which resources and actions an authenticated identity is permitted to access according to security policies, roles, permissions, attributes, and context.

**Question answered:**

> What are you allowed to do?

### Hotel mental model

The reference notes use a useful hotel analogy:

- Showing your reservation and photo ID at the front desk = **authentication**.
- Receiving a room key for your specific room = **authorization**.

The key does not prove your identity by itself. It represents the access granted after identity verification.

### Architecture workflow

```text
Identity Claim
     |
     v
Authentication
     |
     v
Authenticated Identity
     |
     +----> Context / Risk / Device
     |
     v
Authorization Policy
     |
     v
Permission / Entitlement
     |
     v
Resource Access
     |
     v
Audit
```

### Three phases of an identity event

```text
BEFORE AUTHENTICATION
    |
    | IP / device / network / risk signals
    v
DURING AUTHENTICATION
    |
    | credential + MFA + policy
    v
AFTER AUTHENTICATION
    |
    | session / token / authorization
    v
RESOURCE ACCESS
```

---

# Part II — OAuth 2.0 Theory

## 2. OAuth 2.0 Mental Model

OAuth 2.0 is primarily an **authorization framework** for delegated access to protected resources.

It answers:

> What access has been delegated to this client?

It is not, by itself, a standardized user-authentication protocol. OpenID Connect adds the identity layer.

### Core OAuth actors

| Term | Definition |
|---|---|
| Resource Owner | Entity capable of granting access to a protected resource |
| Client | Application requesting access on behalf of the resource owner |
| Authorization Server | Server that handles authorization and issues tokens |
| Resource Server | Server hosting protected resources, commonly an API |
| Authorization Grant | Credential representing the authorization used to obtain an access token |
| Redirect URI | Registered callback destination for the authorization response |
| Access Token | Credential used by the client to access protected resources |
| Scope | Boundary describing requested/delegated access |

### Mental model

```text
Resource Owner
      |
      | grants authorization
      v
Authorization Server
      |
      | access token
      v
Client
      |
      | protected request
      v
Resource Server
```

---

## 3. Front Channel vs Back Channel

### Front channel

The front channel passes through the user's browser/user-agent.

Typical examples:

- Browser redirects
- Authentication requests
- Consent
- Authorization-code delivery

It is more exposed because the browser is part of the communication path.

```text
Client <---- Browser ----> Authorization Server
```

### Back channel

The back channel is direct server-to-server communication.

Typical examples:

- Authorization code exchange
- Token endpoint calls
- Server-side API calls

```text
Client Backend <----------> Authorization Server
```

### Security principle

```text
Browser-mediated channel
        ↓
Minimize sensitive material
        ↓
Short-lived authorization code
        ↓
Direct token exchange
        ↓
Token protected from browser exposure
```

---

# 4. OAuth 2.0 Authorization Code Flow

The reference example uses a user connecting an application to another service.

### Step 1 — Authorization request

The client redirects the browser to the authorization server with information such as:

- Redirect URI
- Response type
- Requested scopes
- Client identifier
- State and other protocol parameters

### Step 2 — Authentication and consent

The authorization server authenticates the resource owner and obtains authorization/consent as required.

### Step 3 — Authorization code

The authorization server redirects the browser to the registered redirect URI with a short-lived authorization code.

### Step 4 — Token exchange

The client backend directly contacts the authorization server and exchanges the code for tokens.

### Step 5 — Resource access

The client sends the access token to the resource server.

### Workflow

```text
                FRONT CHANNEL
User
 |
 v
Client
 |
 | redirect
 v
Authorization Server
 |
 | authenticate + consent
 |
 | authorization code
 v
Browser
 |
 | redirect to registered Redirect URI
 v
Client
 |
 |---------------- BACK CHANNEL ----------------|
 |                                              |
 | authorization code + client authentication   |
 v                                              |
Authorization Server <--------------------------|
 |
 | access token
 v
Client
 |
 | Authorization: Bearer <access_token>
 v
Resource Server / API
 |
 | validate token + authorization
 v
Protected Resource
```

---

# 5. OAuth 2.0 Grant / Flow Types

## Authorization Code

Used for user-delegated authorization.

Modern secure pattern:

```text
Authorization Code + PKCE
```

This is the primary pattern for browser-based and native applications where the client cannot safely keep a static secret.

## Implicit Flow

Historically used for browser applications.

The access token was returned directly through the browser front channel.

The reference notes identify this as an outdated/deprecated security pattern because the token can be exposed to browser history, extensions, and client-side attacks.

Modern architecture uses:

```text
Authorization Code + PKCE
```

instead.

## Resource Owner Password Credentials (ROPC)

The application directly receives the user's username/password.

This is strongly discouraged because it undermines the separation between the application and the identity provider and can interfere with modern authentication controls such as MFA and risk-based access.

## Client Credentials

Used for machine-to-machine authorization where no end user is present.

```text
Workload
   |
   | client authentication
   v
Authorization Server
   |
   | access token
   v
API
```

Typical use cases:

- Service-to-service APIs
- Automation
- CI/CD
- Workload identities

---

# 6. Permissions, Privileges and Scopes

These concepts are related but should not be treated as identical.

### Permission

A granular right to perform an action on a resource.

Examples:

```text
Read Database A
Delete File B
Create User C
```

### Privilege

A collection of permissions or elevated access state, often associated with administrative authority.

Examples:

```text
System Administrator
Database Administrator
Root
Global Administrator
```

### Scope

An OAuth-specific boundary describing delegated access requested or granted to a client.

Example:

```text
User.Read
Mail.Send
read:products
```

A scope does not magically create underlying resource permissions. The resource server ultimately enforces whether the requested operation is allowed.

### Relationship

```text
Identity
   |
   +---- Role / Privilege
             |
             +---- Permissions
                        |
                        +---- Resource actions

OAuth Client
   |
   +---- Requested Scope
             |
             +---- Delegated boundary
                        |
                        v
                  Access Token
                        |
                        v
                  Resource Server
                        |
                        v
             Permission / Policy Decision
```

---

# Part III — OAuth Security

# 7. PKCE

**PKCE = Proof Key for Code Exchange.**

PKCE protects the authorization-code flow, especially for public clients that cannot safely store a static client secret.

### The problem

A public client such as a browser application or native mobile application cannot safely hide a client secret.

If an attacker intercepts the authorization code, the attacker might attempt to exchange it for a token.

### PKCE solution

The client creates a random one-time `code_verifier`.

It derives a `code_challenge`, normally using SHA-256.

```text
code_verifier
      |
      | SHA-256
      v
code_challenge
```

The authorization request carries the challenge.

The token request carries the original verifier.

The authorization server compares:

```text
Hash(code_verifier)
        ==
stored code_challenge
```

Only then is the token issued.

### Full workflow

```text
Client
 |
 | Generate random code_verifier
 |
 | SHA-256
 v
code_challenge
 |
 | authorization request + challenge
 v
Authorization Server
 |
 | stores challenge
 |
 | authorization code
 v
Client
 |
 | authorization code + code_verifier
 v
Authorization Server
 |
 | hash verifier
 |
 | compare with stored challenge
 |
 +---- mismatch ---> DENY
 |
 +---- match ------> ISSUE TOKEN
```

### Key idea

PKCE replaces dependence on a static secret for the authorization-code exchange with a transaction-specific proof.

---

# 8. Confidential vs Public Clients

### Confidential client

An application that can securely protect credentials because it has a trusted backend.

Examples:

- Traditional server-side web application
- Backend service

It can authenticate using a client secret or stronger credential such as a certificate/federated credential.

### Public client

An application that cannot securely protect a static secret.

Examples:

- Single-page browser application
- Native mobile application

Public clients should use PKCE.

### Decision model

```text
Can the application securely protect a credential?
             |
       +-----+-----+
       |           |
      YES          NO
       |           |
       v           v
Confidential     Public
 Client           Client
       |           |
       v           v
Server-side      PKCE
credentials
```

---

# 9. Refresh Tokens

A refresh token is a longer-lived credential used to obtain a new access token without requiring the user to authenticate again.

The architectural purpose is to balance:

```text
Short-lived Access Tokens
          +
Seamless User Experience
          =
Refresh Token
```

### Why access tokens are short-lived

If intercepted, a short lifetime limits the attack window.

### Secure refresh-token practices

- Secure storage
- Rotation
- Reuse detection
- Idle lifetime
- Absolute lifetime
- Server-side revocation
- Token-family revocation where supported

### Rotation concept

```text
Refresh Token R1
      |
      | exchange
      v
Access Token A2
+
Refresh Token R2
      |
      v
R1 becomes invalid
```

If an attacker reuses R1 after rotation:

```text
Reuse detected
      ↓
Possible compromise
      ↓
Revoke token family
      ↓
Force re-authentication
```

---

# Part IV — Tokens and JWT

# 10. ID Token vs Access Token

This is one of the most important IAM boundaries.

| Feature | ID Token | Access Token |
|---|---|---|
| Protocol | OIDC | OAuth 2.0 |
| Purpose | Authentication / identity | Authorization |
| Intended audience | Client application | Resource server/API |
| Format | JWT | Opaque string or JWT |
| Used by | Client | Resource server |
| Typical content | Identity/authentication claims | Authorization/resource claims |

### Golden rule

> **ID Token → client application.**

> **Access Token → resource server/API.**

Do not send an ID token to a backend API as the API's authorization credential.

### Mental model

```text
              Authorization Server
                 /           \
                /             \
          ID Token         Access Token
             |                   |
             v                   v
       Client Application    Resource Server
             |                   |
       "Who is this?"       "What can it do?"
```

### Analogy

The reference uses:

- **ID Token = passport / digital ID card**
- **Access Token = hotel keycard**

The passport tells the application who authenticated.

The keycard represents access to a particular resource.

---

# 11. JWT

**JWT = JSON Web Token.**

The reference defines JWT as a compact, self-contained JSON-based token format whose integrity/authenticity can be verified through a digital signature.

### Structure

```text
HEADER.PAYLOAD.SIGNATURE
```

### Header

Contains metadata such as:

- Token type
- Signing algorithm

Example:

```json
{
  "typ": "JWT",
  "alg": "RS256"
}
```

### Payload

Contains claims.

Common registered claims:

- `iss` — issuer
- `sub` — subject
- `aud` — audience
- `exp` — expiration
- `iat` — issued-at

Other claims can include:

- scopes
- roles
- email
- department
- authentication context

### Signature

The issuer signs the encoded header and payload using a cryptographic key.

The recipient validates the signature using the appropriate verification key.

### Critical distinction

JWT encoding is **not encryption**.

```text
Encoded ≠ Encrypted
```

The payload is normally Base64URL encoded and can be read by someone who possesses the token.

The signature provides integrity/authenticity; it does not make the claims confidential.

---

# 12. JWT Access Token Validation

A resource server should not simply decode a token and trust its contents.

It should validate the security properties required by its policy.

### Issuer (`iss`)

Identifies the authority that issued the token.

```text
Is this token from a trusted issuer?
```

### Subject (`sub`)

Identifies the subject represented by the token.

Use a stable identifier rather than relying on mutable attributes such as email.

### Audience (`aud`)

Identifies the intended recipient/resource.

```text
Token intended for Expense API
             ↓
Payroll API receives it
             ↓
Audience mismatch
             ↓
REJECT
```

### Expiration (`exp`)

Defines when the token becomes invalid.

```text
current_time > exp
        ↓
REJECT
```

### Scope

Defines delegated access boundaries.

```text
scope = read:products
```

A request to delete a product should not be authorized solely because the token is otherwise valid.

### Signature

The resource server validates the signature using the issuer's trusted public key.

### Validation workflow

```text
HTTP Request
     |
     v
Extract Access Token
     |
     v
Parse / Inspect
     |
     +---- issuer valid?
     |
     +---- signature valid?
     |
     +---- audience valid?
     |
     +---- token not expired?
     |
     +---- required scope/role?
     |
     v
Authorization Decision
     |
   +---+---+
   |       |
 ALLOW    DENY
```

### 401 vs 403 mental model

Generally:

- **401 Unauthorized** → authentication/token validation problem.
- **403 Forbidden** → token may be valid, but the caller lacks sufficient authorization.

---

# Part V — URI, URL and URN

# 13. URI / URL / URN

### URI — Uniform Resource Identifier

A general identifier for a resource.

### URL — Uniform Resource Locator

A URI that identifies a resource and provides a way to locate/access it.

Example:

```text
https://example.com/login
```

### URN — Uniform Resource Name

A URI that identifies a resource by a persistent name within a namespace without describing where to retrieve it.

Example:

```text
urn:isbn:9780134685991
```

### Relationship

```text
URI
├── URL
└── URN
```

### Why IAM engineers encounter "Redirect URI"

OAuth terminology uses **Redirect URI** because the callback destination can be more general than a conventional web URL. Native applications may use custom URI schemes.

Example:

```text
https://app.example.com/callback

myapp://callback
```

---

# Part VI — OpenID Connect

# 14. OIDC Theory

**OpenID Connect (OIDC)** is an identity layer built on top of OAuth 2.0.

The fundamental separation is:

```text
OAuth 2.0 → Authorization
OIDC      → Authentication / Identity
```

OIDC provides the client with standardized identity information, primarily through the ID Token.

### OIDC actors

OIDC terminology commonly uses:

- End User
- OpenID Provider (OP)
- Relying Party (RP)

This maps conceptually to the older SAML vocabulary:

```text
SAML:   IdP → SP
OIDC:   OP  → RP
```

### OIDC sign-in model

```text
End User
   |
   v
Relying Party
   |
   | authorization request
   v
OpenID Provider
   |
   | authenticate user
   | obtain authorization
   v
Authorization Response
   |
   v
Client
   |
   +---- ID Token
   |
   +---- Access Token
```

---

# 15. OAuth + OIDC Layered Architecture

```text
┌─────────────────────────────┐
│ OpenID Connect               │
│ Identity / Authentication   │
│ ID Token                    │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│ OAuth 2.0                  │
│ Authorization / Delegation │
│ Access Token               │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│ HTTP / TLS                  │
└─────────────────────────────┘
```

This separation is foundational to modern IAM architecture.

---

# Part VII — SAML Federation

# 16. Identity Provider and Service Provider

### Identity Provider (IdP)

The identity authority authenticates the user and produces a signed assertion/token that another system can trust.

Typical responsibilities:

- Identity management
- Authentication
- MFA
- Authentication policy
- Assertion/token issuance

### Service Provider (SP)

The application or resource the user wants to access.

The SP:

- Trusts the IdP
- Receives the assertion
- Validates it
- Creates its own application session
- Grants access based on its authorization rules

### Trust model

```text
User
 |
 v
Identity Provider
 |
 | signed assertion
 v
Service Provider
 |
 v
Application Session
```

---

# 17. IdP-Initiated SAML Flow

The user begins at the identity portal.

```text
1. User
   |
   v
2. IdP Login
   |
   v
3. IdP Session
   |
   v
4. User selects application
   |
   v
5. IdP creates signed SAML Assertion
   |
   v
6. Browser carries assertion
   |
   v
7. SP ACS URL
   |
   v
8. SP validates signature
   |
   v
9. SP creates application session
```

### Important security point

The assertion passes through the browser.

Therefore:

- Digital signatures matter.
- ACS URL validation matters.
- Trust configuration matters.

### JIT provisioning

A SAML assertion can contain user attributes.

If the SP supports JIT provisioning, it can create a local user profile at first login.

---

# 18. SP-Initiated SAML Flow

The user starts at the application.

```text
1. User
   |
   v
2. Service Provider
   |
   | SAML AuthnRequest
   v
3. Identity Provider
   |
   | authenticate / reuse session
   v
4. Signed SAML Assertion
   |
   v
5. Browser
   |
   v
6. SP ACS URL
   |
   v
7. Assertion validation
   |
   v
8. Application session
```

### RelayState

The SP may use RelayState to preserve the original destination/deep link.

Example:

```text
User requests:
https://app.example.com/reports/123

       ↓

SP → IdP

       ↓

Authentication

       ↓

IdP → SP + assertion + RelayState

       ↓

SP returns user to:
/reports/123
```

---

# Part VIII — Identity Security Fabric

# 19. Passwordless Authentication

Passwordless authentication reduces dependence on shared secrets.

Common technologies include:

- FIDO2/security keys
- Platform biometrics
- Device-bound authenticators
- Other phishing-resistant mechanisms

Mental model:

```text
Traditional:
Password
   ↓
Verifier

Passwordless:
Possession / Device / Biometric
   ↓
Strong Authenticator
   ↓
Identity Provider
```

The objective is not merely convenience; reducing reusable shared secrets can materially reduce credential-phishing exposure.

---

# 20. Single Sign-On

SSO allows a user to authenticate with a central identity authority and access multiple applications without repeatedly entering credentials.

Benefits:

- Better user experience
- Reduced password fatigue
- Centralized authentication policy
- Centralized session/security controls

Architecture:

```text
             Identity Provider
              /     |      \
             /      |       \
           App A   App B    App C
```

SSO also creates concentration risk: compromise of the identity layer can affect many dependent applications.

---

# 21. Federated Identity Management

Federated Identity Management extends trust between separate identity/security domains.

Example:

```text
Organization A
     IdP
      |
      | Federation Trust
      |
      v
Organization B
     Application
```

Federation can allow access without creating separate local passwords in every organization.

---

# 22. Identity Governance and Administration

IGA is the policy-driven framework that helps ensure:

> The right person has the right access for the right reason at the right time.

Core areas:

- Lifecycle management
- Access requests
- Approvals
- Entitlements
- Access reviews
- Certification
- SoD
- Policy enforcement
- Audit evidence

---

# 23. Lifecycle Management — JML

JML means:

```text
Joiner → Mover → Leaver
```

### Joiner

Create identity and baseline access.

### Mover

Change access when responsibilities change.

### Leaver

Remove access when the relationship ends.

### Enterprise workflow

```text
HR / Authoritative Source
          |
          v
    Identity Change
          |
          v
   Lifecycle Engine
          |
    +-----+-----+
    |           |
  Joiner      Mover       Leaver
    |           |           |
    v           v           v
Provision   Recalculate   Revoke
Access        Access       Access
    \           |           /
     \          |          /
          Audit / Report
```

---

# 24. Segregation of Duties

SoD prevents incompatible combinations of access.

Example:

```text
Create Vendor
     +
Approve Vendor Payment
     =
Toxic Combination
```

The objective is to reduce fraud, error, and abuse risk.

SoD can be implemented through:

- Preventive controls
- Approval workflows
- Compensating controls
- Periodic certification
- Detective monitoring

---

# 25. Access Review Certification Campaigns

An access certification campaign formally asks designated reviewers whether users still need their current access.

### Why it exists

It helps enforce:

- Least privilege
- Removal of permission creep
- Compliance
- Accountability

### Campaign lifecycle

```text
1. Define Scope
       ↓
2. Assign Reviewers
       ↓
3. Review Access
       ↓
4. Approve / Revoke / Modify
       ↓
5. Automated Remediation
       ↓
6. Audit Evidence
```

### Reviewer types

- Line manager
- Application owner
- Resource owner
- Privileged-access owner

Good campaigns provide enough context for a meaningful decision.

---

# 26. Privileged Access Management

PAM protects high-impact administrative access.

Typical capabilities:

- Credential vaulting
- Credential rotation
- Session controls
- Approval
- Just-In-Time elevation
- Monitoring
- Recording
- Emergency access

### JIT model

```text
Normal Identity
      |
      v
Request Elevated Access
      |
      v
Policy / Approval
      |
      v
Temporary Privilege
      |
      v
Perform Task
      |
      v
Automatic Expiration
      |
      v
Audit
```

The key security idea is to minimize standing administrative privilege.

---

# 27. Zero Trust and Least Privilege

The reference frames modern access around Zero Trust:

- Do not implicitly trust based on network location.
- Verify identity and context.
- Apply least privilege.
- Assume compromise.
- Contain blast radius.

### Identity-centric Zero Trust

```text
Identity
   +
Device
   +
Location / Network
   +
Application
   +
Resource Sensitivity
   +
Risk
   |
   v
Policy Decision
   |
   +---- Allow
   +---- Step-up
   +---- Deny
```

---

# 28. Identity Threat Protection

Identity threat protection extends security beyond the initial login.

Signals can include:

- Impossible travel
- Anonymous IP
- Compromised credentials
- Abnormal authentication behavior
- Suspicious session behavior

Possible responses:

```text
Risk Detected
     ↓
Increase Authentication
     OR
Revoke Session / Token
     OR
Block Access
     OR
Force Credential Reset
```

---

# Part IX — Access Control Theory

# 29. Access Control Models

Access control governs how subjects interact with protected objects/resources.

### RBAC — Role-Based Access Control

Permissions are associated with roles.

```text
User → Role → Permissions
```

Example:

```text
Security Analyst
      ↓
Read Security Logs
Investigate Alerts
```

### ABAC — Attribute-Based Access Control

Decisions use attributes of:

- Subject
- Resource
- Environment

```text
Subject Attributes
        +
Resource Attributes
        +
Environment
        ↓
      Policy
        ↓
   Permit / Deny
```

### DAC — Discretionary Access Control

The resource owner controls who gets access.

Example:

```text
Document Owner
      ↓
Shares document with User B
```

### MAC — Mandatory Access Control

A central authority defines access through labels/classifications.

Example concept:

```text
Subject Clearance
       +
Resource Classification
       ↓
Central Policy
       ↓
Access Decision
```

Often associated with highly controlled/classified environments.

---

# 30. Permission → Privilege → Role → Scope

Keep these layers separate:

```text
Permission
   ↓
Granular action

Privilege
   ↓
Elevated collection/state of permissions

Role
   ↓
Business/administrative abstraction

Scope
   ↓
OAuth delegated boundary
```

They can interact, but they are not synonyms.

---

# Part X — Enterprise Identity Models

# 31. Workforce Identity

Workforce IAM manages identities such as:

- Employees
- Contractors
- Administrators
- Internal users

Typical priorities:

- Strong authentication
- Least privilege
- JML
- Governance
- SSO
- Privileged access
- Zero Trust

---

# 32. B2B Identity

B2B identity supports external organizational users such as:

- Vendors
- Suppliers
- Consultants
- Partners

### Core priorities

1. Federation / Bring Your Own Identity
2. Strict lifecycle management
3. Time-bound access
4. Governance
5. Host-organization security controls

### B2B architecture

```text
Partner User
     |
     v
Partner IdP
     |
     | Federation
     v
Host Identity Boundary
     |
     | MFA / Risk / Policy
     v
Scoped Application Access
     |
     v
Audit / Governance
```

The partner can authenticate in their home organization while the resource-owning organization still controls authorization.

---

# 33. B2C / CIAM

B2C identity is commonly called **Customer Identity and Access Management (CIAM)**.

Compared with workforce IAM, CIAM strongly emphasizes:

- Frictionless onboarding
- Scalability
- User experience
- Social login
- Passwordless options
- Branding
- Consumer privacy

### Example

A restaurant loyalty application:

```text
Customer
   |
   +---- Continue with Apple
   +---- Continue with Google
   |
   v
CIAM / Identity Provider
   |
   v
OIDC Authentication
   |
   +---- ID Token
   |
   v
Mobile Application
```

The identity architecture must support both security and a low-friction customer experience.

---

# Part XI — Directory and Identity Data

# 34. Universal Identity Directory Concept

A universal identity directory is a centralized identity hub that consolidates identity data from multiple sources into a unified profile.

Typical sources:

```text
HR
 |
AD / Directory
 |
Applications
 |
Contractor Systems
 |
 v
Identity Hub
 |
 +---- Application A
 +---- Application B
 +---- Application C
```

The important vendor-neutral concept is **centralized identity correlation and profile management**, not a particular product name.

---

# 35. Profile Sourcing

Profile sourcing determines where an identity's authoritative attributes originate.

Example:

```text
HR System
   |
   | employee status
   | department
   | manager
   v
Identity Profile
   |
   +---- Application A
   +---- Application B
   +---- Application C
```

A mature architecture defines:

- Attribute ownership
- Source priority
- Mapping
- Data quality
- Change propagation
- Exception handling

Some platforms support multiple sources with priority rules and attribute-level sourcing.

---

# 36. Identity Lifecycle + Profile Sourcing

The two concepts work together:

```text
Authoritative Source
       ↓
Identity Profile
       ↓
Lifecycle Event
       ↓
JML Decision
       ↓
Access Calculation
       ↓
Provision / Modify / Revoke
       ↓
Connected Applications
```

Example:

```text
HR changes:
Department = Finance → HR

        ↓

Identity profile changes

        ↓

Old Finance access becomes ineligible

        ↓

HR access becomes eligible

        ↓

Provision / revoke workflows execute
```

---

# Part XII — Architecture Workflows

# 37. End-to-End Workforce Identity Workflow

```text
                  ┌──────────────────┐
                  │ Authoritative HR │
                  └────────┬─────────┘
                           |
                           v
                  ┌──────────────────┐
                  │ Identity Profile │
                  └────────┬─────────┘
                           |
                    JML / Governance
                           |
             ┌─────────────┼─────────────┐
             v             v             v
          Joiner         Mover         Leaver
             |             |             |
             v             v             v
        Provision     Recalculate      Revoke
             \             |             /
              \            |            /
               v           v           v
                  Access Management
                         |
                         v
                  Authentication
                         |
                         v
                  Authorization
                         |
                         v
                     Resource
                         |
                         v
                    Audit / SOC
```

---

# 38. End-to-End API Authorization Workflow

```text
User / Workload
      |
      v
Client Application
      |
      v
Authorization Server
      |
      +---- Authenticate / Authorize
      |
      v
Access Token
      |
      v
Client
      |
      | Bearer token
      v
Resource Server
      |
      +---- Signature
      +---- Issuer
      +---- Audience
      +---- Expiration
      +---- Scope / Role
      |
      v
Authorization Decision
      |
   +--+--+
   |     |
 Allow  Deny
```

---

# 39. End-to-End SSO Federation Workflow

```text
User
 |
 v
Application
 |
 | "I need authentication"
 v
Identity Provider
 |
 | authenticate + MFA + policy
 v
Signed Assertion / Token
 |
 v
Application
 |
 | validate trust + signature + claims
 v
Local Application Session
 |
 v
Authorization
 |
 v
Resource
```

---

# 40. Identity Security Decision Pipeline

```text
                Identity
                   |
                   v
             Authentication
                   |
                   v
             Device Context
                   |
                   v
             Network Context
                   |
                   v
               Risk Signal
                   |
                   v
             Authentication
                Policy
                   |
                   v
             Authorization
                Policy
                   |
                   v
             Access Decision
             /      |      \
          Allow   Step-Up   Deny
                   |
                   v
                Audit
```

---

# Part XIII — Architect's Mental Models

# 41. Five Questions

For almost every IAM problem ask:

```text
1. WHO is requesting access?

2. HOW do we know who/what it is?

3. WHAT is it allowed to do?

4. WHY does it need that access?

5. HOW do we know the access is still appropriate?
```

---

# 42. Protocol Selection Mental Model

```text
Need user authentication?
        |
        v
      OIDC
        |
        +---- enterprise legacy federation? → SAML may be appropriate

Need delegated API authorization?
        |
        v
     OAuth 2.0

Need machine-to-machine authorization?
        |
        v
Client Credentials / workload identity pattern

Need standardized provisioning?
        |
        v
SCIM
```

This is a conceptual selection model; actual architecture depends on the application, trust model, client type, and security requirements.

---

# 43. Troubleshooting Authentication

```text
Login Failure
     |
     v
Identity exists?
     |
   No → Identity/lifecycle issue
     |
    Yes
     v
Credential valid?
     |
   No → Credential issue
     |
    Yes
     v
MFA successful?
     |
   No → MFA/authenticator issue
     |
    Yes
     v
Policy allows authentication?
     |
   No → Policy/context/risk issue
     |
    Yes
     v
Session/token issued?
     |
   No → Protocol/token issue
     |
    Yes
     v
Application accepts token/assertion?
```

---

# 44. Troubleshooting Authorization

```text
Authentication successful
          |
          v
Identify resource
          |
          v
Identify action
          |
          v
Identify subject
          |
          v
Check role / group / entitlement
          |
          v
Check policy conditions
          |
          v
Check token claims
          |
          v
Check resource-server decision
          |
       +--+--+
       |     |
      200   403
```

---

# 45. API Token Troubleshooting

When an API rejects a token, inspect:

```text
1. Signature
2. Issuer (iss)
3. Audience (aud)
4. Expiration (exp)
5. Not-before / relevant time claims
6. Scope
7. Roles / permissions
8. Token type
9. Required authentication context
10. Resource-server policy
```

Do not treat "JWT decoded successfully" as equivalent to "JWT trusted."

---

# Part XIV — Daily Study Plan

## 30-Day Cycle

| Day | Topic |
|---:|---|
| 1 | Authentication vs Authorization |
| 2 | Identity, Accounts and Credentials |
| 3 | OAuth Actors |
| 4 | Front vs Back Channel |
| 5 | Authorization Code Flow |
| 6 | OAuth Flow Types |
| 7 | Permissions, Privileges, Roles and Scopes |
| 8 | Confidential vs Public Clients |
| 9 | PKCE |
| 10 | Refresh Tokens |
| 11 | JWT Anatomy |
| 12 | JWT Validation |
| 13 | ID Token vs Access Token |
| 14 | URI / URL / URN |
| 15 | OIDC |
| 16 | SAML / Federation |
| 17 | IdP vs SP |
| 18 | SP-Initiated SAML |
| 19 | IdP-Initiated SAML |
| 20 | SSO and Federated Identity |
| 21 | Access Control Models |
| 22 | Zero Trust |
| 23 | IGA |
| 24 | JML / Lifecycle |
| 25 | SoD |
| 26 | PAM / JIT |
| 27 | Identity Threat Protection |
| 28 | B2B / B2C |
| 29 | Profile Sourcing / Identity Data |
| 30 | End-to-End IAM Architecture |

---

# Part XV — Daily Review Questions

After reading a chapter, answer without looking:

### OAuth

1. Who is the Resource Owner?
2. What is the Client?
3. What does the Authorization Server do?
4. What does the Resource Server do?
5. What is a scope?
6. Why is the authorization code short-lived?
7. Why is PKCE needed?
8. What makes a client public?
9. What makes a client confidential?

### Tokens

10. What is the difference between an ID Token and Access Token?
11. Who should consume an ID Token?
12. Who should consume an Access Token?
13. What does `iss` mean?
14. What does `sub` mean?
15. What does `aud` mean?
16. What does `exp` mean?
17. What does `scope` mean?
18. Why isn't a JWT encrypted?

### Federation

19. What is an IdP?
20. What is an SP?
21. What is the difference between SP-initiated and IdP-initiated SAML?
22. What is an ACS URL?
23. What is RelayState?
24. What is JIT provisioning?

### Governance

25. What problem does IGA solve?
26. What is JML?
27. What is an access certification?
28. Why does permission creep happen?
29. What is SoD?
30. Why is PAM different from ordinary IAM?

### Architecture

31. Where is the source of truth?
32. Where is authentication performed?
33. Where is authorization enforced?
34. Where are privileged decisions made?
35. Where is access reviewed?
36. Where are identity events logged?
37. What happens when an identity is terminated?
38. What happens when a user changes jobs?
39. What happens if an access token is stolen?
40. What happens if the identity provider is compromised?

---

# Final Identity Mental Model

```text
                         IDENTITY
                            |
                            v
                      AUTHENTICATION
                            |
             +--------------+--------------+
             |              |              |
          Device          Risk          Context
             |              |              |
             +--------------+--------------+
                            |
                            v
                       AUTHORIZATION
                            |
              +-------------+-------------+
              |             |             |
             RBAC          ABAC        Policy
              |             |             |
              +-------------+-------------+
                            |
                            v
                          ACCESS
                            |
                            v
                       APPLICATION
                         / API
                            |
                            v
                         AUDIT
                            |
                            v
                       GOVERNANCE
                            |
              +-------------+-------------+
              |             |             |
             JML           IGA           PAM
              |             |             |
              +-------------+-------------+
                            |
                            v
                    CONTINUOUS SECURITY
                            |
                            └──────────→ IDENTITY
```

## The principle to remember

> **IAM is not simply login.**

It is the complete discipline of establishing identity, authenticating it, determining what it can access, enforcing least privilege, governing that access throughout its lifecycle, protecting privileged identities, securing machine identities, monitoring identity activity, and continuously reassessing whether access remains appropriate.

---

## Source Note

This notebook is based on the uploaded *Identity Foundations Notes* PDF supplied from the user's Gemini conversation. The theory sections intentionally preserve its terminology and conceptual framing; the workflow diagrams and architect-oriented questions are added as a separate learning layer.

