# SailPoint IdentityIQ (IIQ) --- Enterprise IAM Lab Roadmap

## 25 Hands-On Labs for Enterprise IGA / IAM Engineering

**Target:** Enterprise IAM / IGA Engineer\
**Platform:** SailPoint IdentityIQ (IIQ), version-agnostic across
current IIQ 8.x environments\
**Goal:** Build real enterprise IGA engineering skill: identity data,
application onboarding, correlation, provisioning, JML, RBAC,
governance, SOD, certifications, troubleshooting and architecture.

------------------------------------------------------------------------

## 1. How This Fits Your Existing IAM Labs

You have already worked through Okta administration, Okta REST
API/Postman, SAML SSO with real applications, OAuth/OIDC, Entra IGA
concepts, JML, RBAC, least privilege and IAM troubleshooting. Your
existing Okta roadmap deliberately combines theory, configuration,
validation, intentional failure and troubleshooting rather than
memorizing screens. This IIQ roadmap uses the same approach.
fileciteturn5file5L680-L713

The broader enterprise IAM curriculum already identifies IIQ concepts
such as Identity Cube, connectors, certifications, provisioning, SOD and
role mining. This roadmap turns those concepts into a dedicated 25-lab
enterprise path. fileciteturn5file9L1226-L1284

------------------------------------------------------------------------

# 2. Enterprise Reference Scenario

Use one fictional enterprise across all 25 labs so that every lab builds
on the previous one.

## NorthStar Financial Services

**5,000 employees**

### Authoritative sources

-   Workday / HR CSV
-   Active Directory
-   Contractor source

### Connected applications

-   Active Directory
-   Salesforce
-   ServiceNow
-   Oracle Database
-   SAP
-   Linux / LDAP
-   Custom JDBC application

### Security requirements

-   Joiner / Mover / Leaver automation
-   Least privilege
-   RBAC
-   Segregation of Duties
-   Access certifications
-   Manager and application-owner approvals
-   Privileged-access separation
-   Full auditability
-   Exception handling
-   Reconciliation between IIQ and target systems

### Logical architecture

``` text
                    HR / Authoritative Source
                              |
                              v
                    +--------------------+
                    | SailPoint IIQ      |
                    |                    |
                    | Identity Cube      |
                    | Correlation        |
                    | Roles / Bundles    |
                    | Policies / SOD     |
                    | Workflows          |
                    | Certifications     |
                    | Provisioning       |
                    +----------+---------+
                               |
             +-----------------+------------------+
             |                 |                  |
             v                 v                  v
        Active Directory    Salesforce        ServiceNow
             |                 |                  |
             +--------+--------+------------------+
                      |
                      v
                 Oracle / SAP / LDAP
```

------------------------------------------------------------------------

# 3. Standard Lab Method

Every lab follows:

``` text
WHY → THEORY → DESIGN → BUILD → HAPPY PATH
                       ↓
                BREAK SOMETHING
                       ↓
                  TROUBLESHOOT
                       ↓
                    VERIFY
                       ↓
                   DOCUMENT
                       ↓
               ARCHITECT REVIEW
```

A lab is **not complete** just because the configuration saved. You
should be able to explain the business problem, IIQ object, source of
truth, identity processing, downstream change, evidence and failure
path.

------------------------------------------------------------------------

# 4. 25-Lab Curriculum

## Phase 1 --- IIQ Platform and Identity Foundation

## Lab 01 --- Build the SailPoint IIQ Enterprise Lab

### Objective

Build a safe IIQ development environment and establish an enterprise
operating standard.

### Scenario

NorthStar is deploying IIQ into DEV before moving toward TEST and PROD.

### Tasks

1.  Obtain an authorized IIQ sandbox.
2.  Record IIQ version and patch level.
3.  Identify application server and database components.
4.  Create a lab administrator account.
5.  Document URLs and environment boundaries.
6.  Establish naming conventions.
7.  Establish configuration/change documentation.
8.  Define backup and restore expectations.
9.  Create a change log.
10. Document which features/connectors are available.

### Key concepts

IdentityIQ architecture, Identity Warehouse, Identity Cube,
applications, connectors, tasks, workflows, rules and configuration
objects.

### Outcome

You can explain how an enterprise IIQ platform is structured and
operated.

------------------------------------------------------------------------

## Lab 02 --- Identity Cube and Identity Lifecycle

### Objective

Understand the Identity Cube as the central representation of an
identity.

### Scenario

HR supplies employeeId, name, department, title, manager, location and
employment status. IIQ must maintain a complete identity view and link
downstream accounts.

### Tasks

1.  Create/import test identities.
2.  Inspect Identity Cube attributes.
3.  Add appropriate extended attributes.
4.  Establish manager relationships.
5.  Inspect lifecycle state.
6.  Inspect linked accounts.
7.  Inspect roles and entitlements.
8.  Separate authoritative from non-authoritative attributes.

### Failure exercise

Change an authoritative attribute and determine what detects the change
and when the Identity Cube reflects it.

### Outcome

You can explain:

``` text
Person → Identity Cube → Accounts → Entitlements → Roles
```

------------------------------------------------------------------------

## Lab 03 --- Application Definition and Account Aggregation

### Objective

Onboard a target application and understand aggregation.

### Scenario

NorthStar wants IIQ to govern Active Directory.

### Tasks

1.  Create an application definition.
2.  Configure connector settings.
3.  Configure account schema.
4.  Configure entitlement schema.
5.  Configure aggregation.
6.  Run account aggregation.
7.  Review accounts and entitlements.
8.  Document the application data model.

### Failure exercise

Introduce an incorrect attribute mapping and diagnose the aggregation
result.

### Outcome

You understand:

``` text
Target → Connector → Aggregation → IIQ Application → Identity Cube
```

------------------------------------------------------------------------

## Lab 04 --- Identity Correlation

### Objective

Learn how IIQ decides which target account belongs to which identity.

### Scenario

A Workday identity already has an AD account.

### Tasks

1.  Create an authoritative identity.
2.  Aggregate the AD account.
3.  Configure correlation attributes.
4.  Test employeeId correlation.
5.  Test email/username correlation.
6.  Investigate unmatched accounts.
7.  Investigate ambiguous matches.
8.  Document the correlation hierarchy.

### Failure scenarios

-   Duplicate email
-   Incorrect employeeId
-   Missing employeeId
-   Reused username

### Outcome

You understand why correlation errors can create serious provisioning
and governance problems.

------------------------------------------------------------------------

## Lab 05 --- Identity Refresh and Identity Processing

### Objective

Understand how IIQ recalculates identity state after data changes.

### Scenario

A Finance employee changes department to IT.

### Tasks

1.  Change authoritative attributes.
2.  Run identity refresh.
3.  Observe role membership.
4.  Observe policy evaluation.
5.  Observe lifecycle state.
6.  Observe provisioning implications.
7.  Compare aggregation with identity refresh.

### Mental model

``` text
Aggregation = bring data INTO IIQ
Identity Refresh = process/recalculate identity state INSIDE IIQ
```

### Outcome

You can diagnose a stale Identity Cube even when the source data is
correct.

------------------------------------------------------------------------

## Lab 06 --- Entitlements and Managed Attributes

### Objective

Master entitlement modeling.

### Scenario

Salesforce contains Sales, Finance, Support and Admin values. They must
become governed access items in IIQ.

### Tasks

1.  Aggregate entitlement data.
2.  Review managed attributes.
3.  Configure descriptions and ownership.
4.  Identify privileged entitlements.
5.  Define requestability.
6.  Document entitlement ownership.

### Outcome

You can distinguish:

``` text
Account ≠ Entitlement ≠ Role ≠ Permission
```

------------------------------------------------------------------------

## Lab 07 --- Bundles, Roles and Role Hierarchy

### Objective

Build enterprise RBAC using IIQ bundles.

### Scenario

``` text
Employee
 ├── Finance Employee
 │     └── Finance Reporting
 ├── IT Employee
 │     └── IT Service Access
 └── Manager
       └── Manager Reporting
```

### Tasks

1.  Create business roles.
2.  Create IT roles.
3.  Add entitlements to roles.
4.  Configure inheritance.
5.  Assign roles.
6.  Evaluate effective access.
7.  Remove a role and verify access changes.

### Outcome

You can engineer roles instead of manually assigning entitlements.

------------------------------------------------------------------------

# Phase 2 --- Provisioning and JML

## Lab 08 --- Provisioning Policies

### Objective

Build account creation and entitlement provisioning policies.

### Scenario

A new Finance employee should receive an AD account and Finance access.

### Tasks

1.  Define account provisioning policy.
2.  Define required attributes.
3.  Map identity attributes.
4.  Configure account creation.
5.  Configure entitlement provisioning.
6.  Test provisioning.
7.  Verify the target account.
8.  Verify the IIQ account link.

### Outcome

You understand how IIQ converts identity state into target-system
changes.

------------------------------------------------------------------------

## Lab 09 --- Joiner Automation

### Objective

Build a complete Joiner process.

### Scenario

HR sends:

``` text
Employee ID: NS10025
Department: Finance
Manager: MGR100
Location: Toronto
Status: Active
```

### Tasks

1.  Import the employee.
2.  Correlate/create the Identity Cube.
3.  Evaluate lifecycle state.
4.  Calculate roles.
5.  Trigger provisioning.
6.  Create the AD account.
7.  Assign Finance access.
8.  Verify access.
9.  Review audit evidence.

### Failure exercise

Test a missing-manager condition and document the outcome.

### Outcome

You can explain the Joiner path from HR source to downstream access.

------------------------------------------------------------------------

## Lab 10 --- Mover / Birthright Access

### Objective

Implement access changes when an employee changes job.

### Scenario

A Finance employee moves to IT.

Expected:

``` text
Remove Finance access
        +
Add IT access
        +
Preserve common employee access
```

### Tasks

1.  Change department.
2.  Refresh identity.
3.  Recalculate roles.
4.  Detect removed access.
5.  Detect new access.
6.  Provision changes.
7.  Verify target applications.
8.  Confirm least privilege.

### Outcome

You understand why Movers are more complex than Joiners.

------------------------------------------------------------------------

## Lab 11 --- Leaver and Deprovisioning

### Objective

Implement enterprise termination.

### Scenario

HR changes employmentStatus to Terminated.

Expected:

``` text
Disable accounts
Remove access
Revoke entitlements
Preserve audit history
```

### Tasks

1.  Terminate a test identity.
2.  Trigger lifecycle processing.
3.  Disable AD account.
4.  Remove application access.
5.  Verify account state.
6.  Verify IIQ links.
7.  Review audit records.

### Failure exercise

Simulate a failed deprovisioning task and determine whether IIQ and the
target system agree about termination.

### Outcome

You understand the difference between **identity termination** and
**successful downstream deprovisioning**.

------------------------------------------------------------------------

# Phase 3 --- Workflows, Rules and Automation

## Lab 12 --- IIQ Workflow Fundamentals

### Objective

Understand workflow-driven IAM automation.

### Scenario

A privileged access request requires:

``` text
Requester → Manager Approval → Application Owner Approval
          → Provisioning → Notification
```

### Tasks

1.  Inspect an existing workflow.
2.  Identify variables.
3.  Identify approval steps.
4.  Identify transitions.
5.  Identify provisioning actions.
6.  Execute a test workflow.
7.  Review workflow evidence/logs.

### Outcome

You can read a workflow and explain its execution path.

------------------------------------------------------------------------

## Lab 13 --- Rules: BeanShell / Java Extension Points

### Objective

Understand when IIQ rules are appropriate.

### Scenario

NorthStar needs custom logic that cannot be implemented cleanly through
standard configuration.

### Tasks

1.  Inspect common rule types.
2.  Review inputs and outputs.
3.  Create a small safe lab rule.
4.  Test it.
5.  Review logs.
6.  Compare configuration vs workflow vs rule.

### Design principle

``` text
Configuration first
        ↓
Workflow for process orchestration
        ↓
Rule only when custom logic is genuinely required
```

### Outcome

You understand IIQ's extension model without turning every requirement
into custom code.

------------------------------------------------------------------------

## Lab 14 --- Approval Workflow and Escalation

### Objective

Build enterprise access approval.

### Scenario

Finance access requires manager approval, application-owner approval and
SOD evaluation. If the manager misses the SLA, the request escalates.

### Tasks

1.  Configure approval path.
2.  Configure escalation.
3.  Configure rejection.
4.  Test approval.
5.  Test rejection.
6.  Test timeout/escalation.
7.  Verify provisioning after approval.

### Outcome

You can design approval workflows suitable for production IGA.

------------------------------------------------------------------------

## Lab 15 --- Lifecycle Manager / Access Request

### Objective

Implement user-requested access.

### Scenario

An employee needs temporary Salesforce Finance Reporting access.

### Tasks

1.  Define requestable entitlement.
2.  Configure request workflow.
3.  Submit request.
4.  Approve request.
5.  Provision entitlement.
6.  Verify access.
7.  Record audit evidence.

### Advanced exercise

Add an expiration condition and verify removal.

### Outcome

You understand request-based access versus birthright access.

------------------------------------------------------------------------

# Phase 4 --- Governance, SOD and Certifications

## Lab 16 --- Segregation of Duties Policy

### Objective

Implement preventive/detective SOD controls.

### Scenario

NorthStar must prevent one user from holding both:

``` text
Payment Creation + Payment Approval
```

### Tasks

1.  Define conflicting entitlements/roles.
2.  Create an SOD policy.
3.  Assign policy ownership.
4.  Test a compliant identity.
5.  Test a violating identity.
6.  Review policy violation evidence.
7.  Test an access request that creates a conflict.
8.  Document exception handling.

### Outcome

You can explain SOD as a business-control requirement rather than simply
an IIQ feature.

------------------------------------------------------------------------

## Lab 17 --- SOD Exception and Compensating Control

### Objective

Handle legitimate business exceptions without disabling governance.

### Scenario

A Finance Director needs a temporary conflicting entitlement during
quarter-end.

### Tasks

1.  Create an exception process.
2.  Require business justification.
3.  Require owner approval.
4.  Set an expiration date.
5.  Record compensating control.
6.  Verify the exception is visible in governance reporting.
7.  Test automatic expiry/review.

### Outcome

You learn the enterprise pattern:

``` text
Violation ≠ automatic permanent denial
Violation → controlled exception → approval → expiration → review
```

------------------------------------------------------------------------

## Lab 18 --- Access Certification Campaign

### Objective

Run a real access review campaign.

### Scenario

Quarterly certification requires managers to review direct reports'
access.

### Tasks

1.  Define campaign scope.
2.  Select certification type.
3.  Configure reviewers.
4.  Launch a test campaign.
5.  Approve legitimate access.
6.  Revoke unnecessary access.
7.  Track completion.
8.  Review remediation results.
9.  Capture audit evidence.

### Failure exercise

Have a reviewer reject access and verify the resulting remediation path.

### Outcome

You understand the full loop:

``` text
Access → Review → Decision → Remediation → Evidence
```

------------------------------------------------------------------------

## Lab 19 --- Role Mining and Birthright Role Design

### Objective

Use actual access data to identify candidate roles.

### Scenario

NorthStar has hundreds of manually assigned entitlements and wants to
reduce access complexity.

### Tasks

1.  Analyze entitlement combinations.
2.  Identify common access patterns.
3.  Group identities by department/job function.
4.  Propose candidate roles.
5.  Compare candidate role membership with current access.
6.  Identify over-entitlement.
7.  Validate candidate roles with business owners.
8.  Implement one approved role.

### Outcome

You can explain role mining as a data-driven design activity, not simply
"create roles."

------------------------------------------------------------------------

# Phase 5 --- Enterprise Integration and Operations

## Lab 20 --- Multi-Application Provisioning Pattern

### Objective

Onboard multiple applications using a consistent enterprise pattern.

### Scenario

A Finance employee requires:

``` text
AD → account
Salesforce → Finance profile
ServiceNow → Finance group
Oracle → reporting role
```

### Tasks

1.  Define each application's owner.
2.  Identify authoritative attributes.
3.  Configure application schemas.
4.  Configure aggregation.
5.  Define provisioning policies.
6.  Map roles to entitlements.
7.  Test Joiner provisioning across all four applications.
8.  Test Mover removal/addition.
9.  Test Leaver deprovisioning.

### Outcome

You stop thinking in isolated connectors and start thinking in
**enterprise access chains**.

------------------------------------------------------------------------

## Lab 21 --- Reconciliation and Aggregation Troubleshooting

### Objective

Diagnose the most common IIQ production problem: IIQ and target system
disagree.

### Scenario

Salesforce shows a user with Finance access, but IIQ shows no Finance
entitlement.

### Tasks

1.  Inspect target state.
2.  Inspect application schema.
3.  Run entitlement aggregation.
4.  Compare target and IIQ state.
5.  Inspect identity link.
6.  Run identity refresh.
7.  Determine whether the issue is aggregation, correlation,
    provisioning or refresh.
8.  Document root cause.

### Outcome

You can classify failures instead of blindly rerunning tasks.

------------------------------------------------------------------------

## Lab 22 --- Task Scheduler and Batch Operations

### Objective

Build a production-style operating schedule.

### Scenario

NorthStar needs predictable daily and weekly IAM processing.

### Example schedule

``` text
01:00  Account aggregation
02:00  Entitlement aggregation
03:00  Identity refresh
04:00  Provisioning processing
05:00  Compliance checks
Weekend Role/certification processing
```

### Tasks

1.  Identify required tasks.
2.  Define dependencies.
3.  Configure schedules.
4.  Avoid overlapping heavy tasks.
5.  Review task results.
6.  Create operational runbook.
7.  Define failure/retry procedures.

### Outcome

You understand IIQ as an operational platform, not just a configuration
console.

------------------------------------------------------------------------

## Lab 23 --- IdentityIQ Logging and Production Troubleshooting

### Objective

Become effective at diagnosing IIQ failures from logs and evidence.

### Scenario

A provisioning request says "successful" in the UI but the target
account was not created.

### Tasks

1.  Capture the request ID / task ID.
2.  Trace workflow execution.
3.  Review provisioning logs.
4.  Review connector communication.
5.  Check target-side logs.
6.  Identify authentication vs authorization vs schema vs network
    errors.
7.  Correct the configuration.
8.  Retest.
9.  Document root cause and preventive action.

### Troubleshooting model

``` text
Request
  ↓
Workflow
  ↓
Provisioning Plan
  ↓
Connector
  ↓
Network/API/LDAP/JDBC
  ↓
Target
  ↓
Response
  ↓
IIQ state
```

### Outcome

You can troubleshoot an IAM incident end-to-end.

------------------------------------------------------------------------

## Lab 24 --- Production Change, Deployment and Rollback

### Objective

Practice enterprise IIQ change management.

### Scenario

A new Finance role is approved in DEV and must move to TEST and PROD.

### Tasks

1.  Document the change request.
2.  Identify impacted objects.
3.  Export/package configuration using your organization's supported
    deployment method.
4.  Deploy to TEST.
5.  Execute validation tests.
6.  Capture evidence.
7.  Promote to PROD only after approval.
8.  Create rollback instructions.
9.  Verify production state.

### Enterprise controls

-   Separation of duties
-   Peer review
-   Change ticket
-   Version control where applicable
-   Test evidence
-   Approval before production
-   Rollback plan

### Outcome

You learn how IIQ configuration becomes an enterprise-controlled
deployment rather than an administrator clicking directly in production.

------------------------------------------------------------------------

# Phase 6 --- Enterprise Capstone

## Lab 25 --- End-to-End Enterprise IAM Program

### Objective

Build the complete NorthStar IGA operating model.

### Business scenario

NorthStar acquires a 500-person company. The acquired workforce must be
onboarded into IIQ while preserving business continuity and enforcing
governance.

### Starting state

``` text
Acquired HR
    |
    +-- 500 users
    +-- AD accounts
    +-- Salesforce access
    +-- ServiceNow access
    +-- Oracle access
    +-- Manual legacy permissions
```

### Required design

Build an end-to-end model covering:

1.  Authoritative source.
2.  Identity creation.
3.  Identity correlation.
4.  Identity attributes.
5.  Lifecycle states.
6.  Birthright roles.
7.  Application onboarding.
8.  Account aggregation.
9.  Entitlement aggregation.
10. Provisioning policies.
11. Joiner process.
12. Mover process.
13. Leaver process.
14. Access requests.
15. Manager approvals.
16. Application-owner approvals.
17. SOD policies.
18. SOD exceptions.
19. Access certification.
20. Remediation.
21. Reconciliation.
22. Monitoring.
23. Audit evidence.
24. Change management.
25. Production support runbook.

### Capstone architecture

``` text
                     HR / Acquisition Source
                              |
                              v
                   +-----------------------+
                   | IdentityIQ            |
                   |                       |
                   | Identity Cube         |
                   | Correlation           |
                   | Lifecycle             |
                   | Roles / Entitlements  |
                   | SOD                   |
                   | Certifications        |
                   | Workflows             |
                   | Provisioning          |
                   +-----------+-----------+
                               |
       +-----------------------+-----------------------+
       |                       |                       |
       v                       v                       v
      AD                  Salesforce               ServiceNow
       |                       |                       |
       +-----------------------+-----------------------+
                               |
                               v
                         Oracle / LDAP

       Governance loop:

       Access → Review → Approve/Revoke → Remediate → Audit

       Operations loop:

       Aggregate → Correlate → Refresh → Provision → Reconcile
```

### Capstone failure scenarios

You must intentionally test at least:

-   Duplicate identity.
-   Uncorrelated account.
-   Failed aggregation.
-   Failed provisioning.
-   Missing mandatory attribute.
-   Invalid manager.
-   SOD conflict.
-   Rejected approval.
-   Expired access.
-   Failed deprovisioning.
-   Target/IIQ reconciliation mismatch.

### Final deliverables

Produce:

1.  Enterprise IAM architecture diagram.
2.  Application onboarding matrix.
3.  Identity attribute mapping.
4.  Correlation strategy.
5.  RBAC/role model.
6.  JML workflow diagram.
7.  SOD matrix.
8.  Certification strategy.
9.  Provisioning matrix.
10. Troubleshooting runbook.
11. Production support model.
12. Change/rollback plan.

### Final outcome

You should be able to answer this interview question without relying on
product screenshots:

> **"Walk me through how SailPoint IIQ governs an employee from hire to
> termination across multiple enterprise applications."**

Your answer should naturally cover:

``` text
Authoritative Source
      ↓
Identity Creation / Correlation
      ↓
Identity Cube
      ↓
Lifecycle + Birthright Roles
      ↓
Application Accounts
      ↓
Entitlements
      ↓
Provisioning
      ↓
Access Requests / Approvals
      ↓
SOD / Policy Evaluation
      ↓
Certifications
      ↓
Remediation
      ↓
Reconciliation
      ↓
Audit
```

------------------------------------------------------------------------

# 5. Lab Dependency Map

``` text
01 Platform
   ↓
02 Identity Cube
   ↓
03 Application + Aggregation
   ↓
04 Correlation
   ↓
05 Identity Refresh
   ↓
06 Entitlements
   ↓
07 Roles
   ↓
08 Provisioning
   ↓
09 Joiner
   ↓
10 Mover
   ↓
11 Leaver
   ↓
12 Workflows
   ↓
13 Rules
   ↓
14 Approvals
   ↓
15 Access Requests
   ↓
16 SOD
   ↓
17 Exceptions
   ↓
18 Certifications
   ↓
19 Role Mining
   ↓
20 Multi-App Governance
   ↓
21 Reconciliation
   ↓
22 Operations
   ↓
23 Troubleshooting
   ↓
24 Deployment
   ↓
25 Enterprise Capstone
```

------------------------------------------------------------------------

# 6. Enterprise Skill Matrix

  Lab   Core IIQ skill   Enterprise skill
  ----- ---------------- --------------------------
  01    Platform         Environment architecture
  02    Identity Cube    Identity data model
  03    Application      Connector onboarding
  04    Correlation      Identity matching
  05    Refresh          Identity processing
  06    Entitlements     Access model
  07    Roles            RBAC
  08    Provisioning     Target control
  09    Joiner           JML
  10    Mover            Least privilege
  11    Leaver           Deprovisioning
  12    Workflow         Process orchestration
  13    Rules            Controlled customization
  14    Approval         Governance
  15    Request          Self-service access
  16    SOD              Compliance
  17    Exceptions       Risk management
  18    Certification    Access governance
  19    Role mining      Role engineering
  20    Multi-app        Enterprise integration
  21    Reconciliation   Operational integrity
  22    Tasks            Production operations
  23    Logging          Incident response
  24    Deployment       Change management
  25    Capstone         IAM architecture

------------------------------------------------------------------------

# 7. Recommended Evidence for Every Lab

Create a small evidence package for each completed lab:

``` text
LAB-XX/
├── 01-scenario.md
├── 02-design.md
├── 03-configuration.md
├── 04-test-results.md
├── 05-failure-test.md
├── 06-troubleshooting.md
├── 07-lessons-learned.md
└── screenshots/
```

For production-style learning, also record:

-   What changed?
-   Why was it changed?
-   Who approved it?
-   What identity data was used?
-   What target changed?
-   What evidence proves success?
-   What happens if it fails?
-   How is it rolled back?

------------------------------------------------------------------------

# 8. What You Should Be Able to Explain After Lab 25

## Identity

-   Identity Cube
-   Identity attributes
-   Authoritative source
-   Correlation
-   Identity refresh
-   Lifecycle states

## Access

-   Account
-   Entitlement
-   Role/bundle
-   Birthright access
-   Request-based access
-   Privileged access

## Governance

-   SOD
-   Exceptions
-   Certifications
-   Remediation
-   Ownership
-   Audit evidence

## Operations

-   Aggregation
-   Provisioning
-   Reconciliation
-   Tasks
-   Workflows
-   Rules
-   Logging
-   Deployment

## Architecture

You should be able to design the complete flow:

``` text
HR
 ↓
IIQ Identity Cube
 ↓
Correlation
 ↓
Lifecycle
 ↓
Role Model
 ↓
Entitlements
 ↓
Provisioning
 ↓
Applications
 ↓
Governance
 ↓
Certification
 ↓
Remediation
 ↓
Audit
```

------------------------------------------------------------------------

# 9. Recommended Learning Order with Your Existing Stack

Do not treat IIQ as an isolated product.

``` text
Vendor-neutral IAM theory
        ↓
Entra ID
        ↓
Okta Administration
        ↓
Okta REST APIs
        ↓
SAML / OIDC / OAuth
        ↓
SailPoint IIQ
        ↓
PAM / CyberArk / Delinea
        ↓
Multi-cloud IAM
        ↓
Enterprise IAM Architecture
```

The vendor-neutral foundation is especially important because IAM
concepts such as AuthN/AuthZ, OAuth actors, scopes, roles, permissions
and policy decisions should be understood before mapping them to
product-specific implementations. fileciteturn4file4L15-L20
fileciteturn4file4L101-L123

Your broader enterprise curriculum also separates human identity,
governance, workload identity, multi-cloud IAM, PAM, detection and
vendor-specific platforms. The IIQ labs should therefore be treated as
the **IGA specialization layer**, not as a replacement for the other IAM
domains. fileciteturn4file6L83-L109

------------------------------------------------------------------------

# 10. Final Principle

The goal of these labs is **not**:

> "I know how to configure SailPoint screens."

The goal is:

> **"I can design, implement, troubleshoot and govern enterprise
> identity and access using SailPoint IIQ."**

That distinction is what turns a product administrator into an
enterprise IAM engineer.
