# Microsoft Entra ID — Portal Limitations vs PowerShell & Graph API
## What the portal cannot do — and how to do it anyway
### A living reference document for identity administrators

**Maintainer:** Kamran Arif  
**GitHub:** https://github.com/ds-kamran/ipl-azure-entra-labs  
**Last updated:** 2026-05-30  
**Update frequency:** Weekly — as new limitations are discovered  

---

## Why this document exists

The Entra admin portal covers the most common identity operations well.
Advanced configurations, newer features, and edge cases frequently
ship in the API and PowerShell before the portal catches up.
Sometimes the portal never exposes the option at all.

An administrator who only uses the portal will hit a wall and
assume something is impossible. This document captures what is
actually possible — via PowerShell or Microsoft Graph — when the
portal falls short.

Every entry in this document was discovered through hands-on work,
not from reading documentation. That distinction matters.
Docs describe what features do. This document describes what happens
when you actually try to use them.

---

## How to read each entry

```
Category    — feature area (Groups, Hybrid Identity, Governance etc.)
Impact      — High / Medium / Low — how badly this blocks real work
Limitation  — exactly where the portal stops
Fix         — working PowerShell or Graph command
Why         — real-world consequence if you do not know this
Prevention  — how to avoid the problem entirely
```

---

## Impact key

| Level | Meaning |
|---|---|
| **High** | Blocks a task entirely — portal has no workaround |
| **Medium** | Workaround exists in portal but is incomplete or unreliable |
| **Low** | Portal works but is significantly slower or more error-prone |

---

## Table of Contents

**Administrative Units**
1. [Dynamic AU membership — not available in portal creation wizard](#1-dynamic-administrative-unit-membership)
2. [AU membership type — cannot be changed after creation](#2-au-membership-type-cannot-be-changed-after-creation)
3. [AU restricted management — not a role assignment](#3-au-restricted-management-not-a-role-assignment)
4. [AU-scoped roles — not all built-in roles support AU scoping](#4-au-scoped-roles-not-all-support-au-scoping)

**User & Group Management**
5. [Bulk user creation — UsageLocation omission causes silent licensing failures](#5-bulk-user-creation-usagelocation-silent-failure)
6. [Group-based licensing — blocked on dynamic groups](#6-group-based-licensing-blocked-on-dynamic-groups)
7. [Dynamic group rule — cannot test against all users before saving](#7-dynamic-group-rule-cannot-test-against-all-users)
8. [Role-assignable groups — flag cannot be changed after creation](#8-role-assignable-groups-flag-cannot-change-after-creation)
9. [Guest userType — cannot be changed via portal](#9-guest-usertype-cannot-be-changed-via-portal)

**Hybrid Identity & Cloud Sync**
10. [onPremisesImmutableId — read-only in portal, settable via API](#10-onpremisesimmutableid-read-only-in-portal)
11. [Soft-match migration — no portal workflow exists](#11-soft-match-migration-no-portal-workflow)
12. [Cloud Sync — no full resync option in portal](#12-cloud-sync-no-full-resync-option)
13. [Cloud Sync — quarantine lift location not obvious](#13-cloud-sync-quarantine-lift-not-obvious)
14. [Synced user attributes — greyed out with no explanation](#14-synced-user-attributes-greyed-out)
15. [Password reset for synced users — writeback dependency invisible](#15-password-reset-synced-users-writeback-dependency)

**Privileged Identity Management**
16. [PIM AU-scoped eligible assignments — expiry not enforced consistently](#16-pim-au-scoped-eligible-assignments)

**Conditional Access**
17. [CA policies — cannot be cloned via portal](#17-ca-policies-cannot-be-cloned)
18. [Named locations — no bulk IP range import via portal](#18-named-locations-no-bulk-import)

**Identity Governance**
19. [Lifecycle Workflows — cannot trigger per-user manually from portal](#19-lifecycle-workflows-cannot-trigger-per-user)
20. [Access Reviews — reviewer decision audit trail not fully visible](#20-access-reviews-audit-trail-not-fully-visible)
21. [Entitlement Management — no bulk approval of access requests](#21-entitlement-management-no-bulk-approval)

**Applications & Workload Identity**
22. [Managed Identity — Graph API permissions cannot be assigned via portal](#22-managed-identity-graph-permissions-via-portal)
23. [Service Principal — full permission view not available in portal](#23-service-principal-full-permission-view)
24. [App registration — federated identity credentials have limited portal config](#24-federated-identity-credentials-limited-portal-config)
25. [SCIM provisioning — no clean pause and resume in portal](#25-scim-provisioning-no-clean-pause-resume)

**Monitoring & Compliance**
26. [Audit logs — 30-day retention cannot be extended via portal](#26-audit-logs-retention-cannot-be-extended)
27. [Cross-tenant sync — target attribute mapping restricted in portal](#27-cross-tenant-sync-attribute-mapping-restricted)
28. [PIM for groups — synced and dynamic groups cannot be PIM-enabled](#28-pim-for-groups--synced-and-dynamic-groups-cannot-be-pim-enabled)

---

# ADMINISTRATIVE UNITS

---

## 1. Dynamic Administrative Unit membership

**Category:** Administrative Units  
**Impact:** High  

### Portal behaviour
Entra portal → Roles & admins → Administrative units → Add  
Shows: Name, Description, Restricted management toggle.  
No membership type field. No dynamic rule option.

### The limitation
The AU creation wizard does not expose the Membership type selector.
You cannot create a dynamic AU from the portal.
All users must be manually added — defeating the automation purpose
for any organisation with more than a handful of users per AU.

### The fix

**PowerShell:**
```powershell
Connect-MgGraph -Scopes "AdministrativeUnit.ReadWrite.All"

New-MgDirectoryAdministrativeUnit -BodyParameter @{
    displayName                   = "AU-Engineering"
    description                   = "Automatically populated with Engineering department users"
    membershipType                = "Dynamic"
    membershipRule                = '(user.department -eq "Engineering") and (user.userType -eq "Member")'
    membershipRuleProcessingState = "On"
}
```

**Graph API:**
```
POST https://graph.microsoft.com/v1.0/directory/administrativeUnits
Content-Type: application/json

{
  "displayName": "AU-Engineering",
  "description": "Auto-populates with Engineering department users",
  "membershipType": "Dynamic",
  "membershipRule": "(user.department -eq \"Engineering\") and (user.userType -eq \"Member\")",
  "membershipRuleProcessingState": "On"
}
```

### After creation
Once created via PowerShell or Graph, the AU appears in the portal normally.
The rule is visible and editable. Scoped role assignments work as expected.
Only the initial dynamic configuration cannot come from the portal wizard.

### Why it matters
Without dynamic membership, every new hire in a department requires a
manual AU membership update. At enterprise scale — onboarding hundreds
of users per year across multiple departments — manual AU management
becomes a full-time job and an error-prone one.
Dynamic membership means a user's department attribute change in AD or
Entra automatically moves them into the correct AU scope. Zero manual steps.

### Prevention
Always create AUs via PowerShell when dynamic membership is needed.
Document the decision before creation — you cannot change the
membership type after the fact (see entry 2).

---

## 2. AU membership type cannot be changed after creation

**Category:** Administrative Units  
**Impact:** High  

### Portal behaviour
An existing AU's properties page does not show a membership type field.
There is no convert, change, or update option anywhere in the portal.

### The limitation
Unlike security groups (which can be converted from Assigned to Dynamic
with a warning), Administrative Units have no conversion path.
Assigned stays Assigned. Dynamic stays Dynamic. This is permanent.

### The fix
Delete the AU and recreate it with the correct membership type.
All scoped role assignments on the deleted AU are lost and must be
re-created on the new AU.

```powershell
Connect-MgGraph -Scopes "AdministrativeUnit.ReadWrite.All","RoleManagement.ReadWrite.Directory"

# Step 1 — Document existing scoped role assignments before deleting
$auId = (Get-MgDirectoryAdministrativeUnit -Filter "displayName eq 'AU-ToRecreate'").Id
$scopedRoles = Get-MgDirectoryAdministrativeUnitScopedRoleMember -AdministrativeUnitId $auId
$scopedRoles | ConvertTo-Json | Out-File "C:\Backup\AU-ScopedRoles-Backup.json"
Write-Host "Scoped roles backed up"

# Step 2 — Delete the AU
Remove-MgDirectoryAdministrativeUnit -AdministrativeUnitId $auId

# Step 3 — Recreate with correct membership type
$newAU = New-MgDirectoryAdministrativeUnit -BodyParameter @{
    displayName                   = "AU-ToRecreate"
    membershipType                = "Dynamic"
    membershipRule                = '(user.department -eq "Engineering")'
    membershipRuleProcessingState = "On"
}

Write-Host "Recreated AU: $($newAU.Id)"
Write-Host "Re-assign scoped roles from backup: C:\Backup\AU-ScopedRoles-Backup.json"
```

### Why it matters
Creating 10 AUs with the wrong membership type and then realising
they should be Dynamic means deleting and recreating all 10, plus
re-applying every scoped role assignment. Depending on the organisation
this could mean hours of remediation work and a gap in delegated
admin coverage during the rebuild.

### Prevention
**Ask before creating:** Will membership change frequently based on
user attributes? If yes → Dynamic. If no (e.g. a fixed list of
executives) → Assigned. Decide before running the creation command.

---

## 3. AU restricted management — not a role assignment

**Category:** Administrative Units  
**Impact:** Medium  

### Portal behaviour
The AU creation Properties page has a "Restricted management
administrative unit" toggle (Yes / No).
The Assign roles tab lists administrative roles to delegate.
These look related but serve completely different purposes.

### The limitation
Many administrators look for "Restricted management" in the roles list
and cannot find it. It does not exist there because it is not a role —
it is a protection property set on the AU itself.

### What restricted management actually does
When set to Yes on an AU:
- Only administrators explicitly assigned a scoped role on that AU
  can manage its members
- Even Global Administrators are blocked from managing members
  unless they are explicitly scoped to that AU
- This makes it suitable for protecting highly privileged accounts
  such as executive or break-glass accounts

### How to set it

**During creation (portal):**
Properties tab → Restricted management administrative unit → Yes → Next

**Via PowerShell after creation:**
```powershell
Connect-MgGraph -Scopes "AdministrativeUnit.ReadWrite.All"

Update-MgDirectoryAdministrativeUnit `
    -AdministrativeUnitId "your-au-id" `
    -IsMemberManagementRestricted $true
```

**Verify it is set:**
```powershell
Get-MgDirectoryAdministrativeUnit -AdministrativeUnitId "your-au-id" |
    Select-Object DisplayName, IsMemberManagementRestricted
```

### Why it matters
Placing executive or break-glass accounts in a restricted AU means
a compromised helpdesk account — even one with Password Administrator
tenant-wide — cannot reset those critical accounts.
This is a meaningful additional control layer beyond standard AU scoping.

---

## 4. AU-scoped roles — not all built-in roles support AU scoping

**Category:** Administrative Units  
**Impact:** High — silent security risk  

### Portal behaviour
The Assign roles page on an AU shows a list of roles.
It looks like any role can be scoped to the AU.
The portal does not indicate which roles actually honour the scope.

### The limitation
Several roles appear in the AU role assignment list but silently
become **tenant-wide** when assigned via an AU. The scoping is
not applied. This is not shown as a warning anywhere in the portal.

**Roles that DO support AU scoping:**
- Password Administrator
- Helpdesk Administrator
- User Administrator
- Authentication Administrator
- License Administrator
- Groups Administrator

**Roles that DO NOT support AU scoping (become tenant-wide silently):**
- Global Administrator
- Security Administrator
- Compliance Administrator
- Application Administrator
- Billing Administrator

### The fix
Before assigning any role via an AU, verify it supports scoping:

```powershell
Connect-MgGraph -Scopes "RoleManagement.Read.Directory"

# Check if a specific role supports AU scoping
$roleName = "Security Administrator"
$role = Get-MgRoleManagementDirectoryRoleDefinition `
    -Filter "displayName eq '$roleName'"

Write-Host "Role: $roleName"
Write-Host "Version: $($role.Version)"
# Cross-reference with Microsoft documentation for AU support status
# https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-assign-roles

# List all roles currently assigned to an AU to audit for non-scoped assignments
$auId = (Get-MgDirectoryAdministrativeUnit -Filter "displayName eq 'AU-Engineering'").Id
Get-MgDirectoryAdministrativeUnitScopedRoleMember -AdministrativeUnitId $auId |
    Select-Object RoleId, @{N='RoleName';E={
        (Get-MgRoleManagementDirectoryRoleDefinition -UnifiedRoleDefinitionId $_.RoleId).DisplayName
    }}, RoleMemberId | Format-Table -AutoSize
```

### Why it matters
Assigning "Security Administrator" scoped to an AU — intending to
give a team limited security monitoring rights — actually grants
that person Security Administrator across the entire tenant.
This is a privilege escalation. The admin can now read all security
alerts, manage all security policies, and access data well beyond
their intended scope. The portal gives no warning.

---

# USER & GROUP MANAGEMENT

---

## 5. Bulk user creation — UsageLocation silent failure

**Category:** User provisioning  
**Impact:** High  

### Portal behaviour
Users → Bulk operations → Bulk create provides a CSV template.
The upload succeeds. Users are created. No visible error.

### The limitation
UsageLocation is required for group-based licensing to activate.
If the field is empty in the CSV, users are created without it.
Group licensing then silently fails with "License assignment failed"
— the error message does not mention UsageLocation as the cause.

### The fix
Validate the CSV before upload:

```powershell
$csv = Import-Csv ".\users-to-create.csv"

# Check for missing or invalid usageLocation
$issues = $csv | Where-Object {
    [string]::IsNullOrEmpty($_.usageLocation) -or
    $_.usageLocation.Length -ne 2
}

if($issues.Count -gt 0){
    Write-Host "$($issues.Count) rows have invalid usageLocation:" -ForegroundColor Red
    $issues | Select-Object displayName, userPrincipalName, usageLocation |
        Format-Table -AutoSize
    Write-Host "Fix these rows before uploading. Use 2-letter ISO country code (CA, US, GB etc.)"
} else {
    Write-Host "All rows valid — safe to upload" -ForegroundColor Green
}
```

Fix after the fact for affected users:
```
PATCH https://graph.microsoft.com/v1.0/users/{id}
Body: { "usageLocation": "CA" }
```

Or bulk fix via PowerShell:
```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All"

# Fix all users missing usageLocation who have licenses assigned
Get-MgUser -All -Property Id,UserPrincipalName,UsageLocation,AssignedLicenses |
    Where-Object {
        [string]::IsNullOrEmpty($_.UsageLocation) -and
        $_.AssignedLicenses.Count -gt 0
    } | ForEach-Object {
        Update-MgUser -UserId $_.Id -UsageLocation "CA"
        Write-Host "Fixed: $($_.UserPrincipalName)"
    }
```

### Why it matters
A bulk upload of 500 users where 200 have no UsageLocation results
in 200 silent licensing failures. Users exist, cannot access services,
and the helpdesk receives 200 tickets with no obvious common cause.
Root cause identification takes hours. Prevention takes 30 seconds.

---

## 6. Group-based licensing blocked on dynamic groups

**Category:** Licensing  
**Impact:** High — architectural trap  

### Portal behaviour
Dynamic group → Licenses → Assign → error or no response.
The portal may block the assignment or appear to accept it
then silently fail to apply.

### The limitation
Group-based licensing only works on **Security groups with
Assigned membership**. Dynamic groups cannot hold licenses.
This is a hard platform constraint — not a permissions issue.

### The fix
Use two separate groups serving different purposes:

```powershell
Connect-MgGraph -Scopes "Group.ReadWrite.All"

# Dynamic group — for access control, AU scoping, CA targeting
# (already exists from dynamic group creation)

# Assigned group — for licensing only
$licenseGroup = New-MgGroup -BodyParameter @{
    DisplayName     = "LIC-Engineering-Standard"
    MailEnabled     = $false
    MailNickname    = "LIC-Engineering-Standard"
    SecurityEnabled = $true
    GroupTypes      = @()  # Empty = Assigned security group
}

# Assign license to the assigned group (not the dynamic one)
Set-MgGroupLicense -GroupId $licenseGroup.Id -BodyParameter @{
    AddLicenses    = @(@{SkuId = "your-sku-id"})
    RemoveLicenses = @()
}
```

**Architecture rule:**
```
Dynamic groups  → access control, AU scoping, CA policy targeting
Assigned groups → licensing only
Never mix them
```

### Why it matters
This is the most common architectural mistake in new Entra deployments.
Teams spend hours building a clean dynamic group structure, then discover
they cannot attach licenses and have to retrofit a parallel assigned group
structure. Design for this separation from day one.

---

## 7. Dynamic group rule — cannot test against all users

**Category:** Groups  
**Impact:** Low-Medium  

### Portal behaviour
Groups → your group → Dynamic membership rules → Validate rules  
Lets you test the rule against one user at a time by UPN.
No bulk preview of what the rule would match.

### The limitation
You cannot see "what would this rule match?" across all users
before saving. You save the rule, wait up to 30 minutes for
processing, then check the member count — and find out if
it was right.

### The fix
Simulate the rule result before creating the group:

```powershell
Connect-MgGraph -Scopes "User.Read.All"

# Replace the Where-Object filter to match your intended rule
# Example rule: (user.department -eq "Engineering") and (user.userType -eq "Member")

$wouldMatch = Get-MgUser -All -Property DisplayName,UserPrincipalName,Department,UserType |
    Where-Object {
        $_.Department -eq "Engineering" -and
        $_.UserType   -eq "Member"
    }

Write-Host "Users that would match the rule: $($wouldMatch.Count)"
$wouldMatch | Select-Object DisplayName, UserPrincipalName, Department |
    Format-Table -AutoSize
```

Run this before creating the group. Confirm the count matches
your expectation. Then create the group — the actual count should
match after processing completes.

### Why it matters
Discovering a rule matched 847 users instead of the intended 84
after 30 minutes of processing — then having to correct the rule
and wait another 30 minutes — is avoidable. Pre-validation with
PowerShell takes 10 seconds.

---

## 8. Role-assignable groups — flag cannot change after creation

**Category:** Groups / RBAC  
**Impact:** High — forces delete and recreate  

### Portal behaviour
During group creation: "Azure AD roles can be assigned to this group" toggle.  
After creation: this field is gone from the properties page.
No edit option exists.

### The limitation
The IsAssignableToRole flag is permanent at creation.
A group not created as role-assignable cannot be used for
role assignments — ever. The group must be deleted and recreated.

### The fix
```powershell
Connect-MgGraph -Scopes "Group.ReadWrite.All","RoleManagement.ReadWrite.Directory"

# Create correctly from the start
New-MgGroup -BodyParameter @{
    DisplayName        = "Privileged-Admins"
    MailEnabled        = $false
    MailNickname       = "Privileged-Admins"
    SecurityEnabled    = $true
    IsAssignableToRole = $true   # Cannot be changed after creation
    GroupTypes         = @()
}
```

### Prevention
Any group that might ever need a role assigned to it — now or in the
future — should be created with `IsAssignableToRole = $true` from day one.
There is no operational downside to enabling it proactively.
There is a significant downside to needing it later and having to
rebuild the group.

---

## 9. Guest userType cannot be changed via portal

**Category:** External identity  
**Impact:** Medium  

### Portal behaviour
Guest user profile → userType field is visible but read-only.
No edit button. No change option anywhere in the portal.

### The limitation
B2B invited users arrive as userType = "Guest".
Cross-tenant synced users arrive as userType = "Member".
When the wrong type is provisioned, there is no portal path to correct it.
This matters because Members can hold admin roles; Guests typically cannot.

### The fix
```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All"

Update-MgUser -UserId "user@externaldomain.com" -UserType "Member"

# Verify
(Get-MgUser -UserId "user@externaldomain.com" -Property UserType).UserType
```

### Why it matters
If cross-tenant sync provisions a user as the wrong type, or an external
user needs elevated to Member to receive an admin role assignment, the
portal provides no path. This is a day-one blocker for organisations
running mixed B2B and cross-tenant sync environments.

---

# HYBRID IDENTITY & CLOUD SYNC

---

## 10. onPremisesImmutableId — read-only in portal

**Category:** Hybrid identity  
**Impact:** High — critical for migration scenarios  

### Portal behaviour
User profile → Properties → on-premises section.  
onPremisesImmutableId is visible but the field is read-only.
No portal mechanism exists to set or modify it.

### The limitation
Stamping the ImmutableId on a cloud-only user before Cloud Sync runs
is required for soft-match migration — linking existing cloud identities
to incoming AD objects without creating duplicates.
The portal provides no way to do this.

### The fix

**PowerShell:**
```powershell
Import-Module ActiveDirectory
Connect-MgGraph -Scopes "User.ReadWrite.All"

# Get the AD objectGUID
$adUser     = Get-ADUser -Identity "john.smith" -Properties ObjectGUID
$immutableId = [System.Convert]::ToBase64String($adUser.ObjectGUID.ToByteArray())
Write-Host "ImmutableId: $immutableId"

# Stamp on the cloud user before Cloud Sync runs
Update-MgUser -UserId "john.smith@contoso.com" `
    -OnPremisesImmutableId $immutableId
```

**Graph API:**
```
PATCH https://graph.microsoft.com/v1.0/users/john.smith@contoso.com
Body: { "onPremisesImmutableId": "BASE64_ENCODED_OBJECTGUID" }
```

**Verify the stamp was applied:**
```
GET https://graph.microsoft.com/v1.0/users/john.smith@contoso.com
    ?$select=displayName,onPremisesImmutableId,onPremisesSyncEnabled
```

### Why it matters
Every cloud-to-hybrid migration — acquisitions, organisational
restructuring, environment consolidation — requires this.
Without stamping the ImmutableId before sync runs, Cloud Sync
may create a duplicate account instead of linking to the existing one.
At scale this means duplicate accounts for every migrated user —
each requiring manual cleanup, re-licensing, re-group-assignment,
and MFA re-registration.

---

## 11. Soft-match migration — no portal workflow

**Category:** Hybrid identity  
**Impact:** High — no portal alternative  

### Portal behaviour
There is no "Link this user to an AD object" option anywhere
in the Entra portal. The concept of soft-match does not appear
in any portal screen, blade, or settings page.

### The limitation
Soft-match — the process of linking an existing cloud-only identity
to an incoming AD object during the first Cloud Sync cycle — is
entirely an API and PowerShell concern. The portal provides no
visibility into whether a match occurred, failed, or created a duplicate.

### The fix
Complete workflow:
```powershell
# Step 1 — Export cloud users to migrate
Connect-MgGraph -Scopes "User.Read.All","User.ReadWrite.All"

$cloudUsers = Get-MgUser -All -Property DisplayName,UserPrincipalName,
    OnPremisesSyncEnabled,OnPremisesImmutableId |
    Where-Object {$_.OnPremisesSyncEnabled -ne $true}

$cloudUsers | Export-Csv ".\cloud-users-to-migrate.csv" -NoTypeInformation

# Step 2 — Create matching AD objects (matching UPN is critical)
# Run on domain controller or machine with AD module

# Step 3 — Get objectGUID and stamp ImmutableId for each user
Import-Module ActiveDirectory
$migrationList = Import-Csv ".\cloud-users-to-migrate.csv"

foreach($user in $migrationList){
    $sam = ($user.UserPrincipalName -split "@")[0]
    $adUser = Get-ADUser -Filter {SamAccountName -eq $sam} -Properties ObjectGUID
    if($adUser){
        $immutableId = [System.Convert]::ToBase64String($adUser.ObjectGUID.ToByteArray())
        Update-MgUser -UserId $user.UserPrincipalName -OnPremisesImmutableId $immutableId
        Write-Host "Stamped: $($user.UserPrincipalName)"
    }
}

# Step 4 — Add AD OU to Cloud Sync scope and restart provisioning
# Step 5 — Monitor provisioning logs — look for Action=Update (not Create)
# Action=Update means soft-match succeeded
# Action=Create means a duplicate was created — investigate immediately
```

### Why it matters
Soft-match preserves everything on the existing cloud account:
group memberships, license assignments, MFA registrations, access
package assignments, PIM eligibility, and audit history.
Deleting and recreating accounts loses all of this.
At 1000+ users this difference is weeks of remediation work
versus a pipeline that runs overnight.

---

## 12. Cloud Sync — no full resync option in portal

**Category:** Cloud Sync  
**Impact:** Medium  

### Portal behaviour
Azure AD Connect → Cloud Sync → your configuration → Restart provisioning.  
This sounds like a full resync but is not.
It restarts the provisioning job without clearing the delta watermark.

### The limitation
Delta sync only processes objects that changed since the last cycle.
New attribute mappings do not automatically apply to all existing users.
There is no "force re-evaluate all objects" button in the portal.

### The fix
```powershell
# Option 1 — Restart the agent service (clears local delta state)
# Run on the Windows Server hosting the Cloud Sync agent
Restart-Service "Microsoft Azure AD Connect Provisioning Agent"
# Then click Restart provisioning in the portal

# Option 2 — Touch all in-scope AD users to trigger delta detection
Import-Module ActiveDirectory
Get-ADUser -Filter * -SearchBase "OU=YourSyncedOU,DC=domain,DC=com" `
    -Properties Department |
    ForEach-Object {
        # Writing the same value creates a change event without altering data
        Set-ADUser -Identity $_ -Department $_.Department
    }
Write-Host "Delta touch complete — all users will be re-evaluated on next cycle"
```

### Why it matters
After adding a new attribute mapping — for example, mapping extensionAttribute1
to usageLocation — existing users are not re-processed until they change.
Administrators add the mapping, run a cycle, check a user, see the attribute
is still empty, and conclude the mapping is wrong. It is not wrong. The user
simply has not been delta-processed. This diagnosis takes hours without
knowing this behaviour.

**Cycle duration as a diagnostic signal:**
A cycle completing in under 10 seconds means zero objects were in scope.
This is not success — it is a scope filter problem.

---

## 13. Cloud Sync — quarantine lift not obvious

**Category:** Cloud Sync  
**Impact:** Medium  

### Portal behaviour
When Cloud Sync enters quarantine, a warning banner appears on the
configuration overview. The path to clearing quarantine is not
labelled clearly and many administrators miss it.

### The limitation
Repeatedly restarting the agent service does not clear quarantine.
The quarantine must be explicitly lifted in the portal or via Graph.

### The fix

**Portal path (once you know where to look):**
```
Azure AD Connect → Cloud Sync → your configuration
→ Overview tab → look for quarantine status banner
→ "Resume sync" link within the banner
```

**Graph API (reliable alternative):**
```
POST https://graph.microsoft.com/v1.0/servicePrincipals/{spId}/synchronization/jobs/{jobId}/restart
Content-Type: application/json

{
  "criteria": {
    "resetScope": "Quarantine"
  }
}
```

**Common quarantine triggers:**
- Deletion prevention threshold exceeded (Event ID 12020)
- Connectivity failure lasting beyond grace period (Event ID 12015)
- Invalid credentials on the sync service account
- Scope filter change causing mass deletion

### Why it matters
A quarantined Cloud Sync stops all provisioning and deprovisioning.
New hires do not appear in Entra. Leavers are not deprovisioned.
Every minute in quarantine is a security and compliance gap.
Knowing where the lift is reduces incident resolution from an
hour of trial-and-error to a two-minute fix.

---

## 14. Synced user attributes — greyed out with no explanation

**Category:** Hybrid identity  
**Impact:** Medium  

### Portal behaviour
Synced user → Properties → most attribute fields are greyed out (read-only).
No tooltip, no message, no explanation of why they cannot be edited
or where to make the change instead.

### The limitation
The portal provides no guidance telling administrators that synced user
attributes must be changed in on-premises AD, not in Entra.
Administrators attempt to edit, fail silently or receive a vague error,
and escalate a ticket.

### The fix
Make changes in on-premises AD — they propagate to Entra automatically:
```powershell
Import-Module ActiveDirectory

# Change department in AD (NOT in Entra portal — it will revert on next sync)
Set-ADUser -Identity "john.smith" -Department "Finance"

# Wait for next sync cycle (up to 20 minutes)
# Or trigger immediate sync by restarting the agent:
Restart-Service "Microsoft Azure AD Connect Provisioning Agent"
```

**Attributes editable in Entra even for synced users** (not sync-controlled):
- usageLocation
- mobilePhone
- businessPhones
- preferredLanguage (if not mapped in Cloud Sync)

**Attributes that must be changed in AD** (sync-controlled):
- department
- jobTitle
- mail (if mapped)
- manager
- displayName (if mapped)
- extensionAttributes

### Why it matters
Build and share a reference table with your helpdesk listing which
attributes are AD-owned and which are Entra-owned for synced users.
Without it, helpdesk staff open tickets for "cannot edit user in portal"
that consume IAM team time for something trivially explained once.

---

## 15. Password reset for synced users — writeback dependency invisible

**Category:** Hybrid identity  
**Impact:** Medium  

### Portal behaviour
Users → select synced user → Reset password  
The portal allows the reset. No warning about writeback dependency.
The cloud password changes. The operation appears successful.

### The limitation
For synced users, a cloud password reset only propagates to on-premises AD
if password writeback is explicitly enabled in Cloud Sync configuration.
If writeback is not enabled, the user can sign into cloud apps with
the new password but cannot log into their Windows workstation with it.
The portal does not indicate whether writeback is configured.

### Verify writeback is working
```powershell
Import-Module ActiveDirectory

# After a password reset, check the AD timestamp
Get-ADUser -Identity "john.smith" -Properties pwdLastSet |
    Select-Object Name,
        @{N='PasswordLastSet';E={[datetime]::FromFileTime($_.pwdLastSet)}}

# This timestamp should match the reset time shown in Entra audit logs
# If it does not match — writeback is not working
```

**Enable writeback via portal:**
```
Entra portal → Protection → Password reset
→ On-premises integration
→ Write back passwords to your on-premises directory: Yes
→ Allow users to unlock accounts without resetting their password: Yes
→ Save
```

### Why it matters
Users locked out of their workstation after an admin password reset —
because writeback was not configured — is a P1 incident.
Knowing to verify the pwdLastSet timestamp immediately after a
synced user password reset is the difference between a 2-minute
verification and a 2-hour incident.

---

# PRIVILEGED IDENTITY MANAGEMENT

---

## 16. PIM AU-scoped eligible assignments

**Category:** PIM  
**Impact:** Medium  

### Portal behaviour
PIM → Roles → select role → Add assignments → set scope to an AU.  
Expiry date can be set. The assignment appears to be saved correctly.

### The limitation
For certain role and scope combinations — particularly custom roles
scoped to Administrative Units — the portal does not consistently
enforce or display the expiry date.
The assignment can appear permanent even when a time limit was specified.

### The fix
Create time-bound PIM eligible assignments via Graph to guarantee
expiry is enforced:

```powershell
Connect-MgGraph -Scopes "RoleManagement.ReadWrite.Directory"

$params = @{
    action           = "adminAssign"
    justification    = "Temporary elevated access — change request CR-12345"
    roleDefinitionId = "ROLE_DEFINITION_ID"
    directoryScopeId = "/"
    principalId      = "USER_OBJECT_ID"
    scheduleInfo     = @{
        startDateTime = (Get-Date).ToString("o")
        expiration    = @{
            type        = "afterDateTime"
            endDateTime = "2026-12-31T23:59:59Z"
        }
    }
}

New-MgRoleManagementDirectoryRoleEligibilityScheduleRequest -BodyParameter $params
```

**Verify expiry was applied:**
```
GET https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilitySchedules
    ?$filter=principalId eq 'USER_OBJECT_ID'
    &$select=scheduleInfo,roleDefinitionId,directoryScopeId
```

### Why it matters
Permanent PIM eligibility without expiry violates the principle of
least privilege and most compliance frameworks including SOX and ISO 27001.
If the portal silently drops your expiry setting, you believe access
is time-bounded when it is actually permanent — a governance gap
that external auditors will identify.

---

# CONDITIONAL ACCESS

---

## 17. CA policies — cannot be cloned via portal

**Category:** Conditional Access  
**Impact:** Medium  

### Portal behaviour
Conditional Access → Policies → select a policy.  
Options: Edit, Delete, Enable, Disable.  
No Clone, Duplicate, or Copy option.

### The limitation
Building a policy stack — where multiple policies share a similar
base configuration with minor variations — requires rebuilding
every field from scratch for each policy. No portal shortcut exists.

### The fix
```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.ConditionalAccess","Policy.Read.All"

# Export source policy
$source = Get-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId "SOURCE_POLICY_ID"

# Build clone body
$clone = $source | ConvertTo-Json -Depth 10 | ConvertFrom-Json
$clone.displayName = "$($source.displayName) — Copy"
$clone.state       = "disabled"  # Always create disabled — enable after review

# Remove immutable fields
"id","createdDateTime","modifiedDateTime" | ForEach-Object {
    $clone.PSObject.Properties.Remove($_)
}

# Create the cloned policy
$newPolicy = New-MgIdentityConditionalAccessPolicy -BodyParameter $clone
Write-Host "Cloned policy created: $($newPolicy.Id)"
Write-Host "Review the policy then enable it: Set-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $($newPolicy.Id) -State 'enabled'"
```

**Also useful — export all CA policies as backup before any changes:**
```powershell
Get-MgIdentityConditionalAccessPolicy -All |
    ConvertTo-Json -Depth 10 |
    Out-File ".\CA-Policies-Backup-$(Get-Date -Format 'yyyyMMdd').json"
Write-Host "All CA policies backed up"
```

### Why it matters
An enterprise CA policy stack of 20+ policies — built entirely through
the portal without cloning — takes days and introduces inconsistency
as each policy is manually recreated. Cloning via PowerShell takes
minutes and guarantees the base configuration is identical.
The backup export is also essential — a misconfigured CA policy can
lock administrators out of the tenant instantly.

---

## 18. Named locations — no bulk IP range import

**Category:** Conditional Access  
**Impact:** Low-Medium  

### Portal behaviour
Named locations → new IP ranges location → enter CIDR ranges manually.
One CIDR range per text entry. No import from file.

### The limitation
An organisation with 30 global office locations needs 30+ CIDR ranges
entered manually — one at a time. Error-prone and time-consuming.

### The fix
```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.ConditionalAccess"

# Define ranges in an array or read from CSV
$ipRanges = @(
    "203.0.113.0/24",
    "198.51.100.0/24",
    "192.0.2.0/24"
) | ForEach-Object {
    @{
        "@odata.type" = "#microsoft.graph.iPv4CidrRange"
        "cidrAddress"  = $_
    }
}

New-MgIdentityConditionalAccessNamedLocation -BodyParameter @{
    "@odata.type" = "#microsoft.graph.ipNamedLocation"
    displayName   = "Corporate-Offices-Global"
    isTrusted     = $true
    ipRanges      = $ipRanges
}
```

---

# IDENTITY GOVERNANCE

---

## 19. Lifecycle Workflows — cannot trigger per-user from portal

**Category:** Identity Governance  
**Impact:** Medium  

### Portal behaviour
Identity Governance → Lifecycle Workflows → select workflow → Run history.  
You can view past runs but you cannot trigger the workflow
against a specific user from the portal.

### The limitation
Incident remediation — for example, a leaver workflow that failed
to run for a specific user — requires manual invocation against
that individual. The portal provides no "run now for this user" button.

### The fix
```powershell
Connect-MgGraph -Scopes "IdentityGovernance.ReadWrite.All"

# Find the workflow
$workflow = Get-MgIdentityGovernanceLifecycleWorkflow `
    -Filter "displayName eq 'Employee-Offboarding'"

# Get the target user
$userId = (Get-MgUser -UserId "john.smith@contoso.com").Id

# Trigger manually
Invoke-MgIdentityGovernanceLifecycleWorkflowRun `
    -WorkflowId $workflow.Id `
    -Subjects @(@{
        "@odata.type" = "#microsoft.graph.user"
        id            = $userId
    })

Write-Host "Workflow triggered for john.smith@contoso.com"
```

### Why it matters
During a security incident — "ex-employee account still active" — 
waiting for the next scheduled lifecycle workflow trigger is not
acceptable. Manual invocation via PowerShell is the immediate
remediation path. Without knowing this exists, administrators
resort to manual group removal and account disabling — error-prone
and not auditable as a workflow execution.

---

## 20. Access Reviews — reviewer decision audit trail

**Category:** Identity Governance  
**Impact:** Medium  

### Portal behaviour
Access Reviews → review → Results tab shows decisions and outcomes.  
Who approved or denied is visible. When the decision was made,
the justification provided, and the action taken are not clearly
presented in a format suitable for compliance export.

### The limitation
Compliance frameworks require demonstrating who reviewed what,
what decision was made, when, and what resulted. The portal
summary view does not provide this granularity in a downloadable
audit-ready format.

### The fix
```powershell
Connect-MgGraph -Scopes "AccessReview.Read.All"

$reviewDefinitionId = "YOUR_REVIEW_DEFINITION_ID"
$reviewInstanceId   = "YOUR_REVIEW_INSTANCE_ID"

# Export full decision audit trail
Get-MgIdentityGovernanceAccessReviewDefinitionInstanceDecision `
    -AccessReviewScheduleDefinitionId $reviewDefinitionId `
    -AccessReviewInstanceId $reviewInstanceId |
    Select-Object `
        @{N='ReviewedAt';     E={$_.ReviewedDateTime}},
        @{N='Decision';       E={$_.Decision}},
        @{N='Reviewer';       E={$_.ReviewedBy.UserPrincipalName}},
        @{N='Subject';        E={$_.Principal.UserPrincipalName}},
        @{N='Justification';  E={$_.Justification}},
        @{N='AppliedAt';      E={$_.AppliedDateTime}} |
    Export-Csv ".\AccessReview-AuditTrail-$(Get-Date -Format 'yyyyMMdd').csv" `
        -NoTypeInformation

Write-Host "Audit trail exported for compliance submission"
```

### Why it matters
An auditor asking "show me every access certification decision made
in Q1 2026 with the reviewer name and timestamp" cannot be answered
from the portal alone. This PowerShell export produces the evidence
in a format auditors accept.

---

## 21. Entitlement Management — no bulk approval of requests

**Category:** Identity Governance  
**Impact:** Medium  

### Portal behaviour
Identity Governance → Access packages → Requests → Approve/Deny.  
Each request is approved or denied individually.
No "Approve all" or "Approve selected" option.

### The limitation
During mass access events — open enrollment, system migration,
new office onboarding — hundreds of users may request the same
access package simultaneously. Individual approval of each
is impractical.

### The fix
```powershell
Connect-MgGraph -Scopes "EntitlementManagement.ReadWrite.All"

# Get all pending requests for a specific access package
$packageId = (Get-MgEntitlementManagementAccessPackage `
    -Filter "displayName eq 'Standard-Employee-Access'").Id

$pendingRequests = Get-MgEntitlementManagementAssignmentRequest `
    -Filter "state eq 'pendingApproval' and accessPackage/id eq '$packageId'"

Write-Host "Pending requests to approve: $($pendingRequests.Count)"

foreach($request in $pendingRequests){
    try{
        $approval = Get-MgEntitlementManagementAssignmentRequestApproval `
            -AccessPackageAssignmentRequestId $request.Id

        $stage = Get-MgEntitlementManagementAssignmentRequestApprovalStage `
            -AccessPackageAssignmentRequestId $request.Id `
            -ApprovalId $approval.Id | Select-Object -First 1

        Update-MgEntitlementManagementAssignmentRequestApprovalStage `
            -AccessPackageAssignmentRequestId $request.Id `
            -ApprovalId $approval.Id `
            -ApprovalStageId $stage.Id `
            -ReviewResult "Approve" `
            -Justification "Bulk approved — authorised by IT Manager, ticket INC-12345"

        Write-Host "Approved: $($request.Requestor.UserPrincipalName)" -ForegroundColor Green
    } catch {
        Write-Host "Failed: $($request.Requestor.UserPrincipalName) — $($_.Exception.Message)" `
            -ForegroundColor Red
    }
}
```

---

# APPLICATIONS & WORKLOAD IDENTITY

---

## 22. Managed Identity — Graph API permissions cannot be assigned via portal

**Category:** Workload identity  
**Impact:** High  

### Portal behaviour
Managed identities appear under Enterprise applications in the portal.
The API permissions tab is visible but does not function for assigning
application permissions to managed identities the same way it does
for app registrations.

### The limitation
When a managed identity needs to call Microsoft Graph with application
permissions (User.Read.All, Group.Read.All, etc.), there is no
portal page where you can grant these permissions.
The standard "Grant admin consent" flow that works for app registrations
does not work for managed identities via the portal.

### The fix
```powershell
Connect-MgGraph -Scopes "AppRoleAssignment.ReadWrite.All","Application.Read.All"

# Get the managed identity service principal ID
$miSp = Get-MgServicePrincipal -Filter "displayName eq 'your-managed-identity-name'"

# Get Microsoft Graph service principal
$graphSp = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"

# Find the specific permission to grant
$permission = $graphSp.AppRoles | Where-Object {$_.Value -eq "User.Read.All"}

# Grant the permission
New-MgServicePrincipalAppRoleAssignment `
    -ServicePrincipalId $miSp.Id `
    -BodyParameter @{
        principalId = $miSp.Id
        resourceId  = $graphSp.Id
        appRoleId   = $permission.Id
    }

Write-Host "User.Read.All granted to managed identity: $($miSp.DisplayName)"
```

**Verify permissions were granted:**
```
GET https://graph.microsoft.com/v1.0/servicePrincipals/{managedIdentityId}/appRoleAssignments
```

### Why it matters
Managed identities are the correct authentication pattern for workloads
calling Azure services and Microsoft Graph — no secrets, no expiry,
no rotation burden. If administrators cannot grant the required permissions,
they fall back to client secrets. Secrets expire. Managed identities do not.
This limitation causes teams to choose a less secure option by default.

---

## 23. Service Principal — full permission view not in portal

**Category:** App security  
**Impact:** Medium  

### Portal behaviour
Enterprise applications → your app → Permissions  
Shows admin-consented application permissions.  
Does not show delegated permissions granted by individual users,
historical consent grants, or a consolidated audit view.

### The limitation
The portal permissions tab provides an incomplete picture of
what an application can actually do in your tenant.
User-consented delegated permissions are invisible.

### The fix
```powershell
Connect-MgGraph -Scopes "Application.Read.All","DelegatedPermissionGrant.Read.All"

$appName = "your-application-name"
$sp      = Get-MgServicePrincipal -Filter "displayName eq '$appName'"

Write-Host "=== $appName — Full Permission Audit ===" -ForegroundColor Cyan

# Application permissions (admin-consented)
Write-Host "`n[Application Permissions - Admin Consented]"
Get-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $sp.Id |
    ForEach-Object {
        $resource  = Get-MgServicePrincipal -ServicePrincipalId $_.ResourceId
        $role      = $resource.AppRoles | Where-Object {$_.Id -eq $_.AppRoleId}
        [PSCustomObject]@{
            Resource   = $resource.DisplayName
            Permission = $role.Value
            Granted    = $_.CreatedDateTime
        }
    } | Format-Table -AutoSize

# Delegated permissions (user-consented)
Write-Host "`n[Delegated Permissions - User Consented]"
Get-MgOauth2PermissionGrant -Filter "clientId eq '$($sp.Id)'" |
    Select-Object Scope, ConsentType,
        @{N='GrantedTo';E={if($_.PrincipalId){"Specific user"}else{"All users"}}} |
    Format-Table -AutoSize
```

---

## 24. Federated identity credentials — limited portal configuration

**Category:** Workload identity  
**Impact:** Medium  

### Portal behaviour
App registration → Certificates & secrets → Federated credentials  
Provides templates for common scenarios (GitHub Actions, Kubernetes).
Custom OIDC issuers and complex subject claim patterns have
limited configuration options.

### The limitation
The portal templates force specific subject formats.
Multiple credentials for different environments (main branch,
pull requests, specific environments) must be managed individually.
Custom issuers not in the template list have restricted configuration.

### The fix
```powershell
Connect-MgGraph -Scopes "Application.ReadWrite.All"

$appId = (Get-MgApplication -Filter "displayName eq 'your-app-name'").Id

# Add multiple federated credentials for different scenarios
$credentials = @(
    @{
        name      = "github-main-branch"
        issuer    = "https://token.actions.githubusercontent.com"
        subject   = "repo:org/repo:ref:refs/heads/main"
        audiences = @("api://AzureADTokenExchange")
    },
    @{
        name      = "github-pull-request"
        issuer    = "https://token.actions.githubusercontent.com"
        subject   = "repo:org/repo:pull_request"
        audiences = @("api://AzureADTokenExchange")
    },
    @{
        name      = "github-production-environment"
        issuer    = "https://token.actions.githubusercontent.com"
        subject   = "repo:org/repo:environment:production"
        audiences = @("api://AzureADTokenExchange")
    }
)

foreach($cred in $credentials){
    New-MgApplicationFederatedIdentityCredential `
        -ApplicationId $appId `
        -BodyParameter $cred
    Write-Host "Added: $($cred.name)"
}
```

---

## 25. SCIM provisioning — no clean pause and resume

**Category:** App provisioning  
**Impact:** High during maintenance operations  

### Portal behaviour
Enterprise application → Provisioning → toggle On/Off.  
Turning Off stops provisioning but is not a clean pause.
In some cases it resets the incremental sync watermark,
causing a full re-sync on restart.

### The limitation
There is no "Pause" button in the portal — only On/Off.
A full re-sync after toggling Off then On can send thousands
of provisioning API calls to the target application on restart,
causing rate-limit errors and user access disruption.

### The fix
```powershell
Connect-MgGraph -Scopes "Synchronization.ReadWrite.All","Application.Read.All"

$sp    = Get-MgServicePrincipal -Filter "displayName eq 'your-app-name'"
$jobId = (Get-MgServicePrincipalSynchronizationJob -ServicePrincipalId $sp.Id)[0].Id

# Clean pause — preserves watermark
Invoke-MgPauseServicePrincipalSynchronizationJob `
    -ServicePrincipalId $sp.Id `
    -SynchronizationJobId $jobId
Write-Host "Provisioning paused cleanly — watermark preserved"

# Perform maintenance (update SCIM token, change endpoint, etc.)

# Resume without resetting state
Invoke-MgRestartServicePrincipalSynchronizationJob `
    -ServicePrincipalId $sp.Id `
    -SynchronizationJobId $jobId `
    -Criteria @{ResetScope = "None"}
Write-Host "Provisioning resumed — incremental sync continues from last watermark"
```

### Why it matters
For a SCIM token rotation or endpoint change, an unclean pause
followed by a full re-sync sends provisioning traffic for every
user in scope. At 5000 users with a SCIM endpoint rate limit of
100 requests/minute, that is 50 minutes of continuous API calls
during business hours. Users experience intermittent access failures.
Clean pause/resume is a zero-impact maintenance operation.

---

# MONITORING & COMPLIANCE

---

## 26. Audit logs — 30-day retention not extendable via portal

**Category:** Monitoring  
**Impact:** High — compliance gap  

### Portal behaviour
Monitor → Audit logs — shows events up to 30 days back.  
No setting in the portal to extend this retention period.

### The limitation
SOX, ISO 27001, HIPAA, and most enterprise security policies require
90 days to 1 year of log retention. The portal provides 30 days
(with P1/P2 license — 7 days on free tier) with no extension option.

### The fix
Export logs on a scheduled basis before the 30-day window closes:

```powershell
Connect-MgGraph -Scopes "AuditLog.Read.All"

$exportDate = Get-Date -Format "yyyyMMdd"
$since      = (Get-Date).AddDays(-30).ToString("yyyy-MM-ddTHH:mm:ssZ")

# Sign-in logs
Get-MgAuditLogSignIn -Filter "createdDateTime ge $since" -All |
    Select-Object CreatedDateTime, UserPrincipalName, AppDisplayName,
        IpAddress,
        @{N='ErrorCode';   E={$_.Status.ErrorCode}},
        @{N='Location';    E={$_.Location.City}},
        @{N='DeviceOS';    E={$_.DeviceDetail.OperatingSystem}} |
    Export-Csv ".\SignInLogs-$exportDate.csv" -NoTypeInformation

# Audit logs
Get-MgAuditLogDirectoryAudit -Filter "activityDateTime ge $since" -All |
    Select-Object ActivityDateTime, Category, ActivityDisplayName,
        @{N='InitiatedBy'; E={$_.InitiatedBy.User.UserPrincipalName}},
        @{N='Target';      E={$_.TargetResources[0].UserPrincipalName}},
        Result |
    Export-Csv ".\AuditLogs-$exportDate.csv" -NoTypeInformation

Write-Host "Logs exported: SignInLogs-$exportDate.csv, AuditLogs-$exportDate.csv"
Write-Host "Archive these files to long-term storage immediately"
```

**Long-term solution:**
Configure diagnostic settings to stream logs to Azure Monitor,
Log Analytics, or a SIEM. This is a one-time setup that handles
retention automatically.

### Why it matters
"Show me all admin role activations from the past 6 months"
is a standard auditor request. If logs have not been exported,
everything older than 30 days is gone permanently.
This is a compliance finding — and a security investigation gap
if an incident occurred 45 days ago.

---

## 27. Cross-tenant sync — target attribute mapping restricted in portal

**Category:** Cross-tenant sync  
**Impact:** Medium  

### Portal behaviour
Cross-tenant sync → Attribute mapping → target attribute dropdown  
Shows approximately 20 common attributes.
Many Graph-accessible attributes are not in the dropdown.

### The limitation
Custom extension attributes, less common profile fields, and
organisation-specific metadata cannot be mapped through the
portal dropdown. The attributes exist in Graph but the portal
does not expose them as mapping targets.

### The fix
Edit the synchronization schema directly via Graph API:

```powershell
Connect-MgGraph -Scopes "Synchronization.ReadWrite.All","Application.Read.All"

$sp    = Get-MgServicePrincipal -Filter "displayName eq 'your-cross-tenant-sync-app'"
$jobId = (Get-MgServicePrincipalSynchronizationJob -ServicePrincipalId $sp.Id)[0].Id

# Export current schema for editing
$schema = Get-MgServicePrincipalSynchronizationJobSchema `
    -ServicePrincipalId $sp.Id `
    -SynchronizationJobId $jobId

$schema | ConvertTo-Json -Depth 20 | Out-File ".\sync-schema.json"
Write-Host "Schema exported — edit to add custom attribute mappings"
Write-Host "Then PATCH the schema back via:"
Write-Host "Invoke-MgGraphRequest -Method PATCH -Uri '/v1.0/servicePrincipals/{spId}/synchronization/jobs/{jobId}/schema' -Body (Get-Content .\sync-schema-modified.json -Raw)"
```

### Why it matters
Organisations using cross-tenant sync to consolidate identities
across acquired entities often need to carry classification metadata
(source tenant, department code, cost centre) that is stored in
custom extension attributes. Without Graph schema editing, this
data cannot be mapped and arrives empty in the destination tenant —
breaking downstream automation that depends on those attributes.

---

## Changelog

| Date | # | Change |
|------|---|--------|
| 2026-05-30 | 1–27 | Initial document — all entries from hands-on experience |
| 2026-06-16 | 28 | PIM for groups — synced and dynamic groups excluded from PIM discovery. Discovered during Lab 11 when CSK hybrid groups did not appear in PIM group selector. Architecture pattern added for hybrid environments. |

*Add new entries here each week as new limitations are discovered.*

---

## Contributing

## 28. PIM for groups — synced and dynamic groups cannot be PIM-enabled

**Category:** Privileged Identity Management / Hybrid Identity  
**Impact:** High — blocks JIT elevation on the majority of groups in a hybrid environment  
**Discovered:** Lab 11 — attempting to enable PIM on CSK department groups  

### Portal behaviour

```
Entra portal → Identity Governance → PIM → Groups → Discover groups
```

In the discovery list you will only see cloud-only security groups.
Groups synced from on-premises AD are completely absent from the list.
Dynamic groups are also absent.
No error is shown explaining why — the groups simply do not appear.

### The limitation

PIM can only be enabled on cloud-only, assigned security groups.
Two group types are excluded entirely:

**Synced groups** (`onPremisesSyncEnabled = true`):
Cloud Sync owns membership of these groups — it reads from AD and
writes to Entra. PIM also needs to write membership (to activate and
expire). Two systems cannot both control the same membership object
simultaneously. If PIM activated a user into a synced group and
Cloud Sync then ran a cycle, the membership could be silently overwritten.
Microsoft blocks this conflict by excluding synced groups from PIM entirely.

**Dynamic groups** (`GroupTypes contains DynamicMembership`):
Dynamic group membership is controlled by a rule evaluated against
user attributes. PIM cannot override rule-based membership — activating
a user into a dynamic group would be immediately contradicted if they
do not match the rule. Dynamic groups are therefore excluded.

### Check which of your groups are eligible for PIM

```powershell
Connect-MgGraph -Scopes "Group.Read.All"

Get-MgGroup -All -Property DisplayName,OnPremisesSyncEnabled,
    GroupTypes,IsAssignableToRole |
    Select-Object DisplayName,
        @{N='Synced';     E={$_.OnPremisesSyncEnabled -eq $true}},
        @{N='Dynamic';    E={$_.GroupTypes -contains 'DynamicMembership'}},
        @{N='PIMReady';   E={
            $_.OnPremisesSyncEnabled -ne $true -and
            $_.GroupTypes -notcontains 'DynamicMembership'
        }} |
    Sort-Object PIMReady -Descending |
    Format-Table -AutoSize

# PIMReady = True → cloud-only assigned group → CAN be PIM-enabled
# PIMReady = False → synced or dynamic → CANNOT be PIM-enabled
```

### The fix — create parallel cloud-only groups for PIM scenarios

In a hybrid environment most groups are synced from AD.
The solution is to create dedicated cloud-only groups specifically
for PIM-managed privileged access, alongside your existing synced groups.

```powershell
Connect-MgGraph -Scopes "Group.ReadWrite.All"

# Create a cloud-only PIM-ready group
$pimGroup = New-MgGroup -BodyParameter @{
    DisplayName        = "PIM-Privileged-Admins"
    Description        = "Cloud-only group for JIT privileged access via PIM"
    MailEnabled        = $false
    MailNickname       = "pimprivilegedadmins"
    SecurityEnabled    = $true
    IsAssignableToRole = $true    # Required if group will carry a directory role
    GroupTypes         = @()      # Empty array = assigned security group
}

Write-Host "PIM-ready group created: $($pimGroup.Id)"
Write-Host "Now discover this group in PIM → Groups → Discover groups"
```

**Enable PIM on the cloud-only group:**

```
Entra portal
→ Identity Governance → PIM → Groups
→ Discover groups → find PIM-Privileged-Admins
→ Enable PIM
→ Settings → configure activation policy:
   Require MFA: Yes
   Require justification: Yes
   Max activation duration: 4 hours
   Require approval: Yes
→ Assignments → Add → Eligible → select user → Assign
```

### Architecture pattern for hybrid environments

```
Synced groups (AD-managed)      → day-to-day regular access
Cloud-only PIM groups (Entra)   → privileged JIT access only

User
├── Regular access (always on)
│   ├── CSK-Batting (synced — permanent member via AD)
│   └── F3-USER-GRP (synced — permanent member via AD)
│
└── Privileged access (JIT via PIM)
    └── PIM-CSK-Captains (cloud-only — eligible member)
        ├── Activates for 4 hours on demand
        ├── Requires MFA + justification + approval
        └── Expires automatically
```

The synced groups give regular day-to-day access.
The cloud-only PIM group gives elevated access only when deliberately activated.
Both coexist cleanly because they are separate objects with separate controllers.

### Why IsAssignableToRole matters for PIM groups

If the PIM group will carry a directory role assignment (e.g. User Administrator),
it must be created with `IsAssignableToRole = true`.
This flag cannot be changed after group creation.
If you forget it and create the group without it, you must delete
and recreate — there is no patch.

For PIM groups that only control access package scope, AU delegation,
or CA policy targeting — `IsAssignableToRole` is not required.

### Verify PIM group is working after setup

```powershell
Connect-MgGraph -Scopes "PrivilegedAccess.Read.AzureADGroup"

# List all PIM-enabled groups
Get-MgIdentityGovernancePrivilegedAccessGroupEligibilitySchedule -All |
    Select-Object GroupId, MemberType, Status,
        @{N='Principal'; E={$_.Principal.UserPrincipalName}},
        StartDateTime, EndDateTime |
    Format-Table -AutoSize

# Check active activations right now
Get-MgIdentityGovernancePrivilegedAccessGroupAssignmentScheduleInstance -All |
    Select-Object GroupId, MemberType,
        @{N='Principal'; E={$_.Principal.UserPrincipalName}},
        StartDateTime, EndDateTime |
    Format-Table -AutoSize
```

### Add to your limitations document note

This limitation affects every hybrid Entra deployment.
In environments where all groups originate in AD — which is the
majority of enterprise hybrid setups — no existing group can be
PIM-enabled without creating a parallel cloud-only group structure.
This architectural decision should be made early in the PIM
deployment because retrofitting it requires creating new groups,
re-assigning resources to those groups, and migrating users.

---

Found a portal limitation not listed here?

1. Open an issue on GitHub with label `portal-limitation`
2. Include:
   - What you tried to do in the portal
   - What the portal showed or blocked
   - The PowerShell or Graph fix that worked
   - The real-world impact of not knowing this
3. The limitation will be added with full credit

---

## Reference

| Resource | Link |
|---|---|
| Microsoft Graph API reference | https://learn.microsoft.com/en-us/graph/api/overview |
| Microsoft.Graph PowerShell module | https://learn.microsoft.com/en-us/powershell/microsoftgraph/ |
| Entra admin center | https://entra.microsoft.com |
| Graph Explorer | https://developer.microsoft.com/graph/graph-explorer |
| Entra ID roles that support AU scoping | https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-assign-roles |

---

*Every entry in this document was found by actually hitting the wall —
not by reading documentation.
Docs describe what features do.
This document describes what happens when you try.*

