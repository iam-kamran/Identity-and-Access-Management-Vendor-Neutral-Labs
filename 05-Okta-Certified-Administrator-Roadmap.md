# Okta Certified Administrator Roadmap --- Hands-On Lab & Technical Study Guide

**Author:** Kamran Arif  - https://www.linkedin.com/in/karifa/
**Target certification:** Okta Certified Administrator Performance Exam\
**Primary goal:** Build real administrative skill, not just memorize
exam questions.\
**Recommended duration:** 6 weeks, 5 days/week, 60--90 minutes/day\
**Lab philosophy:** Every topic combines theory, configuration,
validation, troubleshooting, and an enterprise scenario.

> **Important 2026 note:** Okta's current Administrator certification
> preparation material emphasizes performance-based, day-to-day
> operational tasks. The current preparation workshop explicitly
> identifies these areas: Directory Integration, SAML SSO, IdP-initiated
> SAML SSO for Org2Org, Desktop SSO Deployment Federation, Profile
> Sourcing and Write-back, Provisioning, Security Policy and
> Enforcement, Network Zones, Logging/Reporting, Token Management, and
> API Extended Functions.

------------------------------------------------------------------------

## How to Use This Roadmap

For every lab, follow this sequence:

1.  **Read the theory**
2.  **Build the configuration**
3.  **Test the happy path**
4.  **Intentionally break one component**
5.  **Use Okta logs to troubleshoot**
6.  **Document the final design**
7.  **Answer the interview/exam questions**
8.  **Repeat the task without looking at the instructions**

The objective is to develop administrator muscle memory.

------------------------------------------------------------------------

# Lab 0 --- Build the Okta Lab Environment

## Objective

Create a safe Okta environment for experimentation and establish a
repeatable enterprise test model.

## Enterprise Scenario

You are the Okta administrator for:

**NorthStar Financial Services --- 2,000 employees**

The organization uses Okta as its workforce identity platform.

Create a logical model:

``` text
NorthStar Financial Services
                       OKTA LAB TENANT
                             │
             ┌───────────────┼───────────────┐
             │               │               │
          Users            Groups          Admins
             │               │               │
      ┌──────┼──────┐        │         ┌────┼────┐
      │      │      │        │         │    │    │
   Finance   IT     HR       │       Super App Help
                             │       Admin Admindesk
                             │
                 ┌───────────┼───────────┐
                 │           │           │
             NS-Finance    NS-IT       NS-HR
```

## Tasks

-   Obtain an Okta environment suitable for hands-on practice.
-   Record your Okta domain/org name.
-   Create test users.
-   Create test groups.
-   Create a lab naming standard.
-   Install Okta Verify on your test device.
-   Record which Okta Identity Engine features are available in your
    org.
-   Create a lab notebook containing:
    -   configuration
    -   expected result
    -   actual result
    -   troubleshooting notes

## Technical Study

### Official

-   Okta Foundations for Administrators
-   Okta Certified Administrator preparation workshop
-   Okta documentation

### Key concepts

Understand:

-   Okta org
-   Universal Directory
-   Identity Engine
-   Admin Console
-   End-user dashboard
-   Users
-   Groups
-   Applications
-   Policies
-   Authenticators
-   System Log

------------------------------------------------------------------------

# Lab 1 --- Users, Groups & Universal Directory

## Objective

Master the foundation of Okta identity administration.

## Enterprise Scenario

HR is the authoritative source for employee identity data. Okta
Universal Directory maintains identity attributes that are consumed by
downstream applications.

## Tasks

1.  Create 10 test users.
2.  Create:
    -   `NS-Finance`
    -   `NS-IT`
    -   `NS-HR`
    -   `NS-Helpdesk`
3.  Assign users to groups.
4.  Activate and deactivate users.
5.  Reset a user's password.
6.  Add custom profile attributes.
7.  Populate:
    -   employeeId
    -   department
    -   costCenter
    -   manager
8.  Modify attributes.
9.  Create a group rule.
10. Use Okta Expression Language in a group rule.
11. Validate automatic group membership.

## Theory

Understand the difference between:

-   User profile
-   Group
-   Group rule
-   Universal Directory
-   Profile source
-   Profile master
-   Attribute mapping

## Technical Resources

### Official Okta Learning

-   **Define Your Users in Okta** --- Universal Directory, users, custom
    attributes and attribute mappings.
-   **Organize Users with Groups** --- groups, group rules and Okta
    Expression Language.

### Documentation

-   Universal Directory:
    https://help.okta.com/oie/en-us/content/topics/users-groups-profiles/usgp-main.htm
-   Users and profiles:
    https://help.okta.com/oie/en-us/content/topics/users-groups-profiles/usgp-main.htm
-   Groups:
    https://help.okta.com/oie/en-us/content/topics/users-groups-profiles/groups-main.htm
-   Okta Expression Language:
    https://developer.okta.com/docs/reference/okta-expression-language/

------------------------------------------------------------------------

# Lab 2 --- Okta Administrator Roles & Least Privilege

## Objective

Understand Okta administrative RBAC and implement least privilege.

## Enterprise Scenario

The financial institution does not want every IAM engineer to be a Super
Admin.

Model:

``` text
Super Admin
    |
    +-- App Admin
    +-- Group Admin
    +-- Help Desk Admin
    +-- Read/limited administration
```

## Tasks

1.  Review standard administrator roles.
2.  Assign appropriate standard roles.
3.  Create an administrative group.
4.  Assign a role to the group.
5.  Investigate custom administrator roles.
6.  Create a resource set.
7.  Test the administrator's permissions.
8.  Attempt an operation outside the administrator's scope.
9.  Review administrator role assignments.
10. Document the least-privilege design.

## Theory

Understand:

-   Principal
-   Role
-   Resource set
-   Permission
-   Scope
-   Standard role
-   Custom role
-   Group-based admin assignment

## Technical Resources

### Official Okta Learning

-   Define Okta Administrators
-   Assign Standard Administrator Roles
-   Assign Custom Administrator Roles
-   Lab: Define Okta Administrators

### Documentation

-   Roles in Okta:
    https://developer.okta.com/docs/api/openapi/okta-management/guides/roles
-   Okta administrator roles:
    https://help.okta.com/oie/en-us/content/topics/security/administrators-about.htm

## Enterprise Exercise

Design administrator roles for:

  Team                Required access
  ------------------- --------------------------------
  Help Desk           Password/user support
  IAM Operations      User/group/app administration
  Application Admin   Assigned applications
  Security            Read/log investigation
  IAM Manager         Broader administrative control

------------------------------------------------------------------------

# Lab 3 --- SAML SSO

## Objective

Configure and troubleshoot enterprise SAML SSO.

## Enterprise Scenario

NorthStar purchases a SaaS Finance application. Employees must
authenticate through Okta.

``` text
User
 |
 v
Okta
 |
 | SAML Response
 v
FinanceApp
```

## Tasks

1.  Add a SAML application.
2.  Configure:
    -   Single sign-on URL / ACS URL
    -   Audience URI / Entity ID
    -   NameID
    -   attribute statements
3.  Assign the app to a group.
4.  Test SP-initiated SSO.
5.  Test IdP-initiated SSO.
6.  Inspect the SAML response.
7.  Modify a claim/attribute mapping.
8.  Break the ACS URL intentionally.
9.  Diagnose the failure.
10. Restore the correct configuration.

## Theory

You must understand:

-   SAML IdP
-   Service Provider
-   Assertion
-   Response
-   Issuer
-   Audience
-   ACS URL
-   NameID
-   Attribute statements
-   Signature
-   Certificate
-   SP-initiated vs IdP-initiated SSO

## Technical Resources

### Official Okta

-   SAML integrations:
    https://help.okta.com/oie/en-us/content/topics/apps/apps_about_saml.htm
-   Add a SAML application:
    https://help.okta.com/oie/en-us/content/topics/apps/apps_app_integration_wizard_saml.htm
-   Okta SAML troubleshooting:
    https://help.okta.com/oie/en-us/content/topics/apps/apps_saml_troubleshooting.htm

### Recommended protocol theory

-   OASIS SAML Technical Overview:
    https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0-cd-02.html

------------------------------------------------------------------------

# Lab 4 --- Org2Org Federation

## Objective

Configure IdP-initiated SAML federation between Okta organizations.

## Enterprise Scenario

NorthStar acquired another company.

``` text
NorthStar Okta Org
       |
       | Org2Org
       v
Acquired Company Okta Org
```

Users in the subsidiary need access to applications hosted in the parent
organization.

## Tasks

1.  Configure the source Okta org.
2.  Configure the target Okta org.
3.  Establish the IdP relationship.
4.  Configure SAML parameters.
5.  Assign the integration.
6.  Test IdP-initiated access.
7.  Test SP-initiated behavior where applicable.
8.  Investigate the SAML assertion.
9.  Break the relationship intentionally.
10. Troubleshoot.

## Theory

Understand:

-   IdP vs SP roles
-   Federation trust
-   Org2Org
-   SAML assertion flow
-   Routing
-   Application assignment
-   Attribute mapping

## Resources

-   Okta federation documentation:
    https://help.okta.com/oie/en-us/content/topics/identity-providers/oie-about-okta-org2org.htm
-   Okta SAML documentation:
    https://help.okta.com/oie/en-us/content/topics/apps/apps_about_saml.htm

------------------------------------------------------------------------

# Lab 5 --- MFA & Authentication Policies

## Objective

Implement enterprise MFA and authentication policies.

## Enterprise Scenario

NorthStar requires:

-   Strong authentication for employees
-   Stronger authentication for privileged applications
-   Different authentication requirements based on
    group/device/network/risk conditions

## Tasks

1.  Configure authenticators.
2.  Configure Okta Verify.
3.  Configure enrollment policy.
4.  Configure authentication policy.
5.  Configure application sign-in policy.
6.  Create a rule for a privileged group.
7.  Require stronger authentication for FinanceApp.
8.  Test normal users.
9.  Test privileged users.
10. Review System Log events.

## Theory

Understand:

-   Authentication vs authorization
-   Authenticator
-   Enrollment policy
-   Authentication policy
-   App sign-in policy
-   Rule priority
-   Factor requirements
-   Passwordless authentication
-   Phishing-resistant authentication
-   Risk and device conditions

## Resources

-   Authentication policies:
    https://help.okta.com/oie/en-us/content/topics/identity-engine/policies/about-policies.htm
-   App sign-in policy rules:
    https://help.okta.com/oie/en-us/content/topics/identity-engine/policies/add-app-sign-on-policy-rule.htm
-   Okta Verify:
    https://help.okta.com/oie/en-us/content/topics/identity-engine/authenticators/okta-verify-main.htm
-   Phishing-resistant authenticators:
    https://help.okta.com/oie/en-us/content/topics/identity-engine/authenticators/about-phish-resistant-authenticators.htm

------------------------------------------------------------------------

# Lab 6 --- Network Zones

## Objective

Use network context in authentication decisions.

## Enterprise Scenario

NorthStar corporate offices use known public IP ranges.

``` text
Corporate Network -> trusted
Internet           -> stronger authentication
Blocked network    -> denied
```

## Tasks

1.  Create an IP zone.
2.  Define trusted IP addresses.
3.  Create a dynamic zone if available.
4.  Configure authentication policy conditions.
5.  Require stronger authentication outside trusted zones.
6.  Test from an appropriate network.
7.  Review System Log.
8.  Investigate a blocked request.
9.  Correlate network-zone information with the policy evaluation.

## Theory

Understand:

-   IP zones
-   Dynamic zones
-   Trusted networks
-   Policy evaluation
-   IP conditions
-   Limitations of IP-based trust
-   Why location should not be the only security control

## Resources

-   Network zones:
    https://help.okta.com/oie/en-us/content/topics/security/network/network-zones.htm
-   Network zone troubleshooting:
    https://help.okta.com/oie/en-us/content/topics/security/network/ip-service-category-syslog.htm
-   Authentication policy rules:
    https://help.okta.com/oie/en-us/content/topics/identity-engine/policies/add-app-sign-on-policy-rule.htm

------------------------------------------------------------------------

# Lab 7 --- Active Directory Integration

## Objective

Build and troubleshoot Okta + Active Directory integration.

## Enterprise Scenario

NorthStar has an existing Windows AD environment and wants Okta to
provide cloud SSO while maintaining AD as an identity source.

``` text
Active Directory
       |
       | Okta AD Agent
       v
     Okta
       |
       +-- SaaS Apps
       +-- SSO
       +-- Lifecycle
```

## Tasks

1.  Build a small Windows AD lab if available.
2.  Install the Okta AD Agent.
3.  Configure AD integration.
4.  Configure delegated authentication.
5.  Import users.
6.  Import groups.
7.  Configure profile sourcing.
8.  Test authentication.
9.  Test user lifecycle.
10. Review import logs.
11. Break the agent connection and troubleshoot.

## Theory

Understand:

-   AD Agent
-   Delegated authentication
-   Import
-   Profile source
-   Profile master
-   Lifecycle source
-   Agent architecture
-   Firewall/network considerations
-   High availability

## Resources

-   Configure AD import/account settings:
    https://help.okta.com/oie/en-us/content/topics/directory/ad-agent-configure-import.htm
-   Okta AD Agent:
    https://help.okta.com/oie/en-us/content/topics/directory/ad-agent-main.htm
-   Active Directory integration:
    https://help.okta.com/oie/en-us/content/topics/directory/ad-agent-main.htm

------------------------------------------------------------------------

# Lab 8 --- Profile Sourcing & Attribute Mapping

## Objective

Understand how Okta determines the authoritative source of identity
data.

## Enterprise Scenario

AD is authoritative for employee identity attributes.

``` text
AD
 |
 | profile source
 v
Okta
 |
 | attribute mappings
 v
SaaS Apps
```

## Tasks

1.  Make AD the profile source.
2.  Import users.
3.  Map:
    -   firstName
    -   lastName
    -   email
    -   department
    -   employeeId
4.  Change an AD attribute.
5.  Run/import synchronization.
6.  Verify the Okta profile.
7.  Verify downstream application data.
8.  Test lifecycle changes.
9.  Investigate what happens when the source user is deactivated.

## Theory

Master:

-   Profile sourcing
-   Profile source
-   Profile master
-   Attribute mapping
-   Write-back
-   Lifecycle state synchronization
-   Upstream vs downstream application

## Resources

-   Profile sourcing:
    https://help.okta.com/oie/en-us/content/topics/users-groups-profiles/usgp-profile-sourcing.htm
-   Add provisioned users / profile sourcing:
    https://help.okta.com/oie/en-us/content/topics/provisioning/lcm/lcm-about-user-management.htm
-   AD profile sourcing:
    https://help.okta.com/oie/en-us/content/topics/directory/ad-agent-configure-import.htm

------------------------------------------------------------------------

# Lab 9 --- Provisioning & Deprovisioning

## Objective

Automate application account lifecycle through Okta.

## Enterprise Scenario

When an employee joins Finance:

``` text
Okta
 |
 +-- Assign Finance group
 |
 +-- Assign FinanceApp
 |
 +-- Provision account
 |
 +-- Push attributes
```

When the employee leaves:

``` text
Okta
 |
 +-- Remove assignment
 |
 +-- Deactivate downstream account
 |
 +-- Remove application access
```

## Tasks

1.  Select a free/trial SaaS app or suitable test SCIM endpoint.
2.  Configure provisioning.
3.  Configure Create Users.
4.  Configure Update User Attributes.
5.  Configure Deactivate Users.
6.  Assign application through group membership.
7.  Create a test user.
8.  Verify downstream provisioning.
9.  Modify the user.
10. Verify update.
11. Remove the application assignment.
12. Verify deprovisioning.
13. Inspect System Log.

## Theory

Understand:

-   SCIM
-   Push vs pull
-   Upstream/downstream
-   Provisioning
-   Deprovisioning
-   Application assignment
-   Group-based provisioning
-   Lifecycle state
-   Attribute synchronization

## Resources

-   Okta provisioning overview:
    https://help.okta.com/en-us/content/topics/provisioning/lcm/con-okta-prov.htm
-   Add provisioned users:
    https://help.okta.com/oie/en-us/content/topics/provisioning/lcm/lcm-about-user-management.htm
-   On-premises provisioning:
    https://help.okta.com/en-us/content/topics/provisioning/opp/opp-main.htm
-   SCIM standard: https://www.rfc-editor.org/rfc/rfc7644

------------------------------------------------------------------------

# Lab 10 --- JML Lifecycle Management

## Objective

Build a realistic Joiner-Mover-Leaver process.

## Enterprise Scenario

### Joiner

``` text
HR
 |
 v
Okta
 |
 +-- Group assignment
 +-- App assignment
 +-- MFA
 +-- Provisioning
```

### Mover

``` text
Finance
   |
   v
IT
   |
   +-- Remove Finance access
   +-- Add IT access
   +-- Update attributes
```

### Leaver

``` text
HR termination
       |
       v
Okta deactivation
       |
       +-- Remove app assignments
       +-- Deprovision
       +-- Revoke access
```

## Tasks

1.  Create a new employee.
2.  Add appropriate group membership.
3.  Assign application access.
4.  Provision downstream application.
5.  Change department.
6.  Remove old access.
7.  Add new access.
8.  Deactivate user.
9.  Verify downstream deprovisioning.
10. Investigate the lifecycle events in System Log.

## Theory

Understand:

-   Joiner
-   Mover
-   Leaver
-   Source of truth
-   Profile sourcing
-   Group rules
-   Application assignment
-   Provisioning
-   Deprovisioning
-   Access removal

## Resources

-   Okta Lifecycle Management:
    https://help.okta.com/oie/en-us/content/topics/lcm/lcm-main.htm
-   Provisioning:
    https://help.okta.com/en-us/content/topics/provisioning/lcm/con-okta-prov.htm
-   Profile sourcing:
    https://help.okta.com/oie/en-us/content/topics/users-groups-profiles/usgp-profile-sourcing.htm

------------------------------------------------------------------------

# Lab 11 --- System Log & IAM Security Operations

## Objective

Learn to investigate identity events using Okta System Log.

## Enterprise Scenario

A Finance employee reports:

> "I cannot access FinanceApp."

Security asks you to determine why.

## Investigation Flow

``` text
User
 |
 v
Time
 |
 v
System Log
 |
 +-- Authentication event
 +-- Policy evaluation
 +-- Application assignment
 +-- MFA
 +-- Network
 |
 v
Root Cause
 |
 v
Remediation
```

## Tasks

Investigate:

-   Successful login
-   Failed login
-   MFA failure
-   SSO failure
-   Application assignment
-   Application unassignment
-   Admin change
-   Provisioning failure
-   Deprovisioning
-   Network zone block

Practice System Log filters.

Examples:

``` text
eventType eq "application.user_membership.remove"
```

and:

``` text
eventType eq "security.request.blocked"
```

## Theory

Understand:

-   Actor
-   Target
-   Event type
-   Client
-   IP chain
-   Request
-   Policy evaluation
-   Authentication event
-   Application event
-   Audit trail

## Resources

-   System Log:
    https://help.okta.com/oie/en-us/content/topics/reports/reports_syslog.htm
-   Monitoring and reports:
    https://help.okta.com/oie/en-us/Content/Topics/Reports/monitoring-and-reports.htm
-   System Log filters:
    https://help.okta.com/en-us/Content/Topics/Reports/syslog-filters.htm
-   Common System Log filters:
    https://help.okta.com/en-us/content/topics/reports/common-syslog-filters.htm

------------------------------------------------------------------------

# Lab 12 --- API & Token Management

## Objective

Use Okta APIs to perform controlled identity administration and
understand API authentication.

## Enterprise Scenario

IAM operations wants to automate identity reporting.

``` text
PowerShell / Postman
       |
       v
Okta API
       |
       +-- Users
       +-- Groups
       +-- Apps
       +-- Roles
```

## Tasks

### Part A --- API token

1.  Create a dedicated lab service account.
2.  Create an API token.
3.  Query users.
4.  Query groups.
5.  Query applications.
6.  Review token privilege inheritance.
7.  Deactivate the token.

### Part B --- OAuth 2.0

1.  Create an OAuth client/service app.
2.  Assign required admin privileges.
3.  Request scoped access.
4.  Obtain an access token.
5.  Call an Okta API.
6.  Compare API token vs OAuth 2.0.

## Critical theory

Know:

``` text
API Token
   |
   +-- broad privilege
   +-- inherited from creator
   +-- SSWS
```

versus:

``` text
OAuth 2.0
   |
   +-- scoped
   +-- bearer access token
   +-- better granularity
   +-- preferred for new API integrations
```

## Resources

-   Core Okta API:
    https://developer.okta.com/docs/reference/core-okta-api/
-   API reference: https://developer.okta.com/docs/api
-   Create API token:
    https://developer.okta.com/docs/guides/create-an-api-token/main/
-   OAuth API access:
    https://developer.okta.com/docs/guides/set-up-oauth-api/main/
-   Okta Management API:
    https://developer.okta.com/docs/api/openapi/okta-management/guides/overview
-   Postman/API testing: https://developer.okta.com/docs/reference/rest/

## Security note

Okta currently recommends scoped OAuth 2.0 access tokens over the
proprietary SSWS API-token mechanism for API access.

------------------------------------------------------------------------

# Lab 13 --- Desktop SSO / Kerberos Federation

## Objective

Understand Okta Desktop SSO in a Windows/AD environment.

## Enterprise Scenario

NorthStar wants employees who authenticate to the corporate Windows
domain to access Okta applications with reduced authentication friction.

``` text
Windows Device
      |
      | Kerberos
      v
Active Directory
      |
      v
Okta
      |
      v
SaaS Application
```

## Tasks

1.  Review Okta Desktop SSO architecture.
2.  Understand AD integration requirements.
3.  Understand Kerberos/SPN requirements.
4.  Configure Desktop SSO in a lab if your Okta environment supports it.
5.  Test authentication.
6.  Review System Log.
7.  Troubleshoot a failed Desktop SSO request.

## Theory

Master:

-   Kerberos
-   SPN
-   AD Agent
-   IWA/Desktop SSO
-   Federation
-   Integrated authentication
-   SAML relationship

## Resources

-   Okta Desktop SSO documentation:
    https://help.okta.com/oie/en-us/content/topics/directory/desktop-sso-main.htm
-   Okta AD Agent:
    https://help.okta.com/oie/en-us/content/topics/directory/ad-agent-main.htm
-   Kerberos overview:
    https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview

------------------------------------------------------------------------

# Lab 14 --- Troubleshooting Capstone

## Objective

Demonstrate that you can diagnose broken Okta configurations rather than
simply create them.

This is the most important lab before the performance exam.

------------------------------------------------------------------------

## Scenario 1 --- SAML Failure

Symptoms:

> User reaches Okta but application rejects the SAML response.

Investigate:

``` text
ACS URL
Entity ID
Issuer
NameID
Certificate
Assignment
Attribute mapping
```

------------------------------------------------------------------------

## Scenario 2 --- User Not Provisioned

Symptoms:

> User is assigned to the application but no account exists downstream.

Investigate:

``` text
Assignment
   |
Provisioning settings
   |
Credentials
   |
API connectivity
   |
Profile mapping
   |
System Log
```

------------------------------------------------------------------------

## Scenario 3 --- Wrong Application Access

Symptoms:

> Finance user has access to HRApp.

Investigate:

``` text
User
 |
Group membership
 |
Group rules
 |
Application assignment
 |
Policy
```

------------------------------------------------------------------------

## Scenario 4 --- MFA Not Triggered

Investigate:

``` text
Authenticator enrollment
       |
Authentication policy
       |
App sign-in policy
       |
Rule priority
       |
User/group
       |
Network/device/risk
```

------------------------------------------------------------------------

## Scenario 5 --- AD Import Failure

Investigate:

``` text
AD Agent
 |
Network
 |
Credentials
 |
Import settings
 |
Profile sourcing
 |
Import logs
```

------------------------------------------------------------------------

## Scenario 6 --- API Automation Failure

Investigate:

``` text
Authentication
 |
Token
 |
OAuth scope
 |
Admin role
 |
Endpoint
 |
HTTP response
```

------------------------------------------------------------------------

# Final Capstone --- NorthStar IAM Environment

After completing all labs, build the following integrated environment.

``` text
                       HR / AD
                         |
                         | Identity Source
                         v
                  +--------------+
                  |     Okta     |
                  |   Workforce  |
                  +------+-------+
                         |
       +-----------------+------------------+
       |                 |                  |
       v                 v                  v
    Groups           Policies           Admin RBAC
       |                 |                  |
       v                 v                  v
 Applications        MFA/Zones          Least Privilege
       |
       +----------+-----------+
       |          |           |
      SAML       OIDC       SCIM
       |          |           |
       v          v           v
     SaaS       Apps       Lifecycle
```

Your final environment should demonstrate:

-   Users
-   Groups
-   Group rules
-   Universal Directory
-   Custom attributes
-   Admin roles
-   SAML SSO
-   Org2Org
-   MFA
-   Authentication policies
-   Network Zones
-   AD integration
-   Profile sourcing
-   Provisioning
-   Deprovisioning
-   JML
-   System Log investigation
-   API access
-   Token management
-   Desktop SSO concepts

------------------------------------------------------------------------

# Six-Week Study Schedule

## Week 1 --- Okta Foundations

### Day 1

Lab 0 --- Environment

### Day 2

Lab 1 --- Users

### Day 3

Lab 1 --- Groups and Group Rules

### Day 4

Lab 2 --- Administrator Roles

### Day 5

Review + rebuild Labs 1--2 without instructions

------------------------------------------------------------------------

## Week 2 --- SSO & Federation

### Day 6

SAML theory

### Day 7

Lab 3 --- SAML configuration

### Day 8

SAML troubleshooting

### Day 9

Lab 4 --- Org2Org

### Day 10

SAML/Org2Org rebuild from scratch

------------------------------------------------------------------------

## Week 3 --- Security

### Day 11

Lab 5 --- MFA

### Day 12

Authentication Policies

### Day 13

Lab 6 --- Network Zones

### Day 14

Policy troubleshooting

### Day 15

Security review

------------------------------------------------------------------------

## Week 4 --- Directory & Lifecycle

### Day 16

Lab 7 --- AD integration

### Day 17

AD import

### Day 18

Lab 8 --- Profile sourcing

### Day 19

Lab 9 --- Provisioning

### Day 20

JML

------------------------------------------------------------------------

## Week 5 --- Operations

### Day 21

System Log

### Day 22

Troubleshooting

### Day 23

API fundamentals

### Day 24

OAuth/API token management

### Day 25

Desktop SSO/Kerberos

------------------------------------------------------------------------

## Week 6 --- Performance Preparation

### Day 26

Rebuild SAML lab

### Day 27

Rebuild provisioning/JML lab

### Day 28

Rebuild MFA/policy lab

### Day 29

Troubleshooting Capstone

### Day 30

Full simulated Administrator Performance Exam

------------------------------------------------------------------------

# Exam Readiness Checklist

You should be able to perform each task without step-by-step
instructions:

-   [ ] Create users
-   [ ] Activate/deactivate users
-   [ ] Create groups
-   [ ] Create group rules
-   [ ] Use Okta Expression Language
-   [ ] Add custom profile attributes
-   [ ] Map attributes
-   [ ] Assign applications
-   [ ] Assign administrator roles
-   [ ] Explain least privilege
-   [ ] Configure SAML SSO
-   [ ] Troubleshoot SAML
-   [ ] Configure Org2Org
-   [ ] Configure MFA/authenticators
-   [ ] Configure authentication policies
-   [ ] Configure Network Zones
-   [ ] Integrate AD
-   [ ] Import users
-   [ ] Configure profile sourcing
-   [ ] Configure provisioning
-   [ ] Configure deprovisioning
-   [ ] Explain JML
-   [ ] Investigate System Log
-   [ ] Search System Log using event filters
-   [ ] Understand API tokens
-   [ ] Use OAuth 2.0 for Okta APIs
-   [ ] Understand Desktop SSO
-   [ ] Troubleshoot identity failures

------------------------------------------------------------------------

# High-Value Theory Topics to Master

Do not simply learn where to click.

Be able to explain **why** each component exists.

## Identity

-   Universal Directory
-   Profile source
-   Profile master
-   Attribute mapping
-   Lifecycle state

## Authentication

-   Password
-   MFA
-   Authenticators
-   Authentication policies
-   Sign-on policies
-   Risk
-   Device context
-   Network zones

## Federation

-   SAML
-   IdP
-   SP
-   Assertions
-   Certificates
-   Org2Org
-   Desktop SSO
-   Kerberos

## Authorization

-   Groups
-   Application assignment
-   Admin roles
-   Resource sets
-   Least privilege

## Lifecycle

-   JML
-   Provisioning
-   SCIM
-   Deprovisioning
-   Profile sourcing
-   Group rules

## Operations

-   System Log
-   Reports
-   Audit events
-   Troubleshooting
-   API
-   OAuth
-   API tokens

------------------------------------------------------------------------

# Official Certification Preparation

Start with Okta's current Administrator Certification Preparation
Workshop:

https://learning.okta.com/administrator-cert-prep-webinar

It explicitly identifies the Administrator Performance Exam areas used
in this roadmap.

Also use:

-   Okta Foundations for Administrators:
    https://learning.okta.com/okta-foundations-for-administrators
-   Define Your Users in Okta:
    https://learning.okta.com/path/define-your-users-in-okta
-   Organize Users with Groups:
    https://learning.okta.com/path/organize-users-with-groups
-   Define Okta Administrators:
    https://learning.okta.com/path/define-okta-administrators

------------------------------------------------------------------------

# Important Study Strategy for You

You already have strong Microsoft Entra/IAM experience.

Therefore, don't learn Okta as if you're starting IAM from zero.

Use this comparison model:

  ------------------------------------------------------------------------
  IAM capability          Microsoft Entra         Okta
  ----------------------- ----------------------- ------------------------
  Directory               Entra ID                Universal Directory

  User                    Entra User              Okta User

  Group                   Entra Group             Okta Group

  SSO                     Enterprise Application  Application Integration

  SAML                    Entra SAML IdP          Okta SAML IdP

  OIDC                    App Registration        OIDC Application

  MFA                     Authentication Methods  Authenticators

  Conditional access      Conditional Access      Authentication/Sign-On
                                                  Policies

  Admin RBAC              Entra Roles             Okta Admin Roles

  AD integration          Entra Connect           Okta AD Agent

  Provisioning            SCIM                    Okta Provisioning

  Lifecycle               Lifecycle Workflows     Lifecycle Management

  Logs                    Sign-in/Audit Logs      System Log

  API                     Microsoft Graph         Okta APIs
  ------------------------------------------------------------------------

The **IAM concept stays the same; the implementation model changes.**

That is the fastest way for an experienced Entra engineer to become
productive in Okta.

------------------------------------------------------------------------

# Official Sources Used for This Roadmap

1.  Okta Certified Administrator Preparation Workshop --- current
    Administrator Performance Exam topics.
2.  Okta Learning --- Define Your Users in Okta.
3.  Okta Learning --- Organize Users with Groups.
4.  Okta Learning --- Define Okta Administrators.
5.  Okta Foundations for Administrators.
6.  Okta Help --- System Log.
7.  Okta Help --- Authentication Policies.
8.  Okta Help --- AD Integration.
9.  Okta Help --- Profile Sourcing.
10. Okta Help --- Provisioning.
11. Okta Developer --- Core API.
12. Okta Developer --- OAuth API access.
13. Okta Developer --- API token management.
14. Okta Developer --- Roles.

**Note:** Okta documentation changes frequently. When a lab's UI or
feature name differs from this guide, use the current Okta Help/Learning
documentation as the authoritative reference. Avoid relying on old
"Classic Engine" tutorials unless the lab specifically requires
understanding a legacy environment.
