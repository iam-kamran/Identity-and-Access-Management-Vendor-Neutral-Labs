# IAM First Responder Labs — Unusual Sign-Ins & Identity-Based Threats
### Kamran Arif — Enterprise-Level PowerShell & Microsoft Graph
### Microsoft Entra ID P2 | Microsoft Sentinel | Log Analytics

**Author:** Kamran Arif  - https://www.linkedin.com/in/karifa/


> **Before You Start — Environment Setup**
>
> These labs assume:
> - Microsoft Entra ID P2 licence active
> - Microsoft Graph PowerShell SDK installed (`Install-Module Microsoft.Graph -Scope CurrentUser`)
> - Log Analytics workspace connected to Entra ID (from Lab 10 in the CI/CD labs)
> - Global Reader or Security Reader role for investigation queries
> - Security Administrator role for containment actions
>
> **Golden Rule:** Always query before you act. Never contain before you understand scope.
> Disabling the wrong account in a bank at 9am is a worse outcome than a 10-minute delay.

---

## Table of Contents

- [Lab 1 — Build Your Investigation Toolkit](#lab-1--build-your-investigation-toolkit)
- [Lab 2 — Detect and Investigate Impossible Travel](#lab-2--detect-and-investigate-impossible-travel)
- [Lab 3 — Detect and Respond to MFA Fatigue Attacks](#lab-3--detect-and-respond-to-mfa-fatigue-attacks)
- [Lab 4 — Hunt for Credential Stuffing Across the Tenant](#lab-4--hunt-for-credential-stuffing-across-the-tenant)
- [Lab 5 — Detect App Registration Persistence Mechanisms](#lab-5--detect-app-registration-persistence-mechanisms)
- [Lab 6 — Investigate Suspicious PIM Activations](#lab-6--investigate-suspicious-pim-activations)
- [Lab 7 — Token Theft Detection and Containment](#lab-7--token-theft-detection-and-containment)
- [Lab 8 — Full Incident Response Simulation](#lab-8--full-incident-response-simulation)

---

## Lab 1 — Build Your Investigation Toolkit

### What You Will Learn
- How to connect to Microsoft Graph with the correct scopes for security investigation
- How to build reusable PowerShell functions for common IR queries
- How to structure an investigation session so nothing is missed

### Why This Matters for BoC
A BoC technical panel will ask "walk me through what you do first when an identity alert fires." This lab builds the repeatable process you describe in that answer. Having named functions ready means your first 5 minutes on an incident are structured, not ad-hoc.

---

### Step 1 — Connect with Security Investigation Scopes

Create a file called `Connect-IRSession.ps1`:

```powershell
# Connect-IRSession.ps1
# Run this at the start of every investigation session
# These scopes cover read-only investigation — no containment actions yet

function Connect-IRSession {
    param(
        [Parameter(Mandatory)]
        [string]$TenantId
    )

    Write-Host "=== IAM First Responder Session ===" -ForegroundColor Cyan
    Write-Host "Connecting to tenant: $TenantId" -ForegroundColor Yellow
    Write-Host "Scopes: Read-only investigation + Security signals" -ForegroundColor Yellow

    Connect-MgGraph -TenantId $TenantId -Scopes @(
        # Sign-in and audit investigation
        "AuditLog.Read.All",
        "Directory.Read.All",

        # Identity Protection risk signals
        "IdentityRiskyUser.Read.All",
        "IdentityRiskEvent.Read.All",

        # User and group investigation
        "User.Read.All",
        "Group.Read.All",

        # Application investigation (App Reg credential changes)
        "Application.Read.All",

        # Policy investigation
        "Policy.Read.All",

        # Security alerts from Identity Protection
        "SecurityEvents.Read.All"
    )

    # Confirm connection
    $context = Get-MgContext
    Write-Host "`n=== Connected ===" -ForegroundColor Green
    Write-Host "Account:  $($context.Account)" -ForegroundColor Green
    Write-Host "Tenant:   $($context.TenantId)" -ForegroundColor Green
    Write-Host "Scopes:   $($context.Scopes.Count) granted" -ForegroundColor Green
    Write-Host "`nReady to investigate. Remember: QUERY before you ACT.`n" -ForegroundColor Cyan
}

Connect-IRSession -TenantId "<your-tenant-id>"
```

---

### Step 2 — Build the Core Investigation Functions

Create `IR-Functions.ps1` — your reusable toolkit:

```powershell
# IR-Functions.ps1
# Source this file at the start of every investigation
# . .\IR-Functions.ps1

# ============================================================
# FUNCTION 1 — Get sign-in summary for a user
# ============================================================
function Get-UserSignInSummary {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [int]$DaysBack = 7
    )

    Write-Host "`n=== Sign-In Summary: $UserPrincipalName ===" -ForegroundColor Cyan
    Write-Host "Window: Last $DaysBack days`n"

    $startDate = (Get-Date).AddDays(-$DaysBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    $signIns = Get-MgAuditLogSignIn `
        -Filter "userPrincipalName eq '$UserPrincipalName' and createdDateTime ge $startDate" `
        -All `
        -Property "createdDateTime,ipAddress,location,appDisplayName,
                   clientAppUsed,deviceDetail,status,conditionalAccessStatus,
                   riskLevelDuringSignIn,riskLevelAggregated,riskEventTypes"

    if (-not $signIns) {
        Write-Host "No sign-ins found in the last $DaysBack days." -ForegroundColor Yellow
        return
    }

    # Summary statistics
    $successful = $signIns | Where-Object { $_.Status.ErrorCode -eq 0 }
    $failed = $signIns | Where-Object { $_.Status.ErrorCode -ne 0 }
    $riskySignIns = $signIns | Where-Object {
        $_.RiskLevelDuringSignIn -in @("medium","high")
    }
    $uniqueIPs = $signIns | Select-Object -ExpandProperty IpAddress -Unique
    $uniqueLocations = $signIns |
        Select-Object -ExpandProperty Location |
        Select-Object -ExpandProperty CountryOrRegion -Unique

    Write-Host "Total sign-ins:      $($signIns.Count)"
    Write-Host "Successful:          $($successful.Count)" -ForegroundColor Green
    Write-Host "Failed:              $($failed.Count)" -ForegroundColor Red
    Write-Host "Risky (Med/High):    $($riskySignIns.Count)" -ForegroundColor $(
        if ($riskySignIns.Count -gt 0) { "Red" } else { "Green" }
    )
    Write-Host "Unique IP addresses: $($uniqueIPs.Count)"
    Write-Host "Countries seen:      $($uniqueLocations -join ', ')"

    Write-Host "`n--- Recent Sign-Ins (last 10) ---"
    $signIns |
        Sort-Object CreatedDateTime -Descending |
        Select-Object -First 10 |
        ForEach-Object {
            $status = if ($_.Status.ErrorCode -eq 0) { "✓" } else { "✗" }
            $risk = if ($_.RiskLevelDuringSignIn -ne "none") {
                " [RISK: $($_.RiskLevelDuringSignIn)]"
            } else { "" }
            Write-Host "$status $($_.CreatedDateTime) | $($_.IpAddress) | $($_.Location.CountryOrRegion) | $($_.AppDisplayName)$risk"
        }
}

# ============================================================
# FUNCTION 2 — Get all changes made by a user (audit trail)
# ============================================================
function Get-UserAuditTrail {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [int]$HoursBack = 48
    )

    Write-Host "`n=== Audit Trail: $UserPrincipalName ===" -ForegroundColor Cyan
    Write-Host "Window: Last $HoursBack hours`n"

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    $auditLogs = Get-MgAuditLogDirectoryAudit `
        -Filter "initiatedBy/user/userPrincipalName eq '$UserPrincipalName' and activityDateTime ge $startDate" `
        -All `
        -Property "activityDateTime,activityDisplayName,result,targetResources,initiatedBy"

    if (-not $auditLogs) {
        Write-Host "No audit events found — account made no directory changes in this window." -ForegroundColor Green
        return
    }

    Write-Host "Changes made: $($auditLogs.Count)" -ForegroundColor $(
        if ($auditLogs.Count -gt 10) { "Red" } else { "Yellow" }
    )

    $auditLogs | Sort-Object ActivityDateTime -Descending | ForEach-Object {
        $target = $_.TargetResources[0].DisplayName
        Write-Host "$($_.ActivityDateTime) | $($_.ActivityDisplayName) | Target: $target | Result: $($_.Result)"
    }
}

# ============================================================
# FUNCTION 3 — Get current risk state for a user
# ============================================================
function Get-UserRiskState {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName
    )

    Write-Host "`n=== Risk State: $UserPrincipalName ===" -ForegroundColor Cyan

    $user = Get-MgUser -Filter "userPrincipalName eq '$UserPrincipalName'" `
            -Property "id,displayName,accountEnabled,userPrincipalName"

    if (-not $user) {
        Write-Host "User not found." -ForegroundColor Red
        return
    }

    Write-Host "Account enabled: $($user.AccountEnabled)" -ForegroundColor $(
        if ($user.AccountEnabled) { "Green" } else { "Red" }
    )

    # Get risky user details
    $riskyUser = Get-MgRiskyUser -Filter "userPrincipalName eq '$UserPrincipalName'"

    if ($riskyUser) {
        Write-Host "Risk level:      $($riskyUser.RiskLevel)" -ForegroundColor $(
            switch ($riskyUser.RiskLevel) {
                "high"   { "Red" }
                "medium" { "Yellow" }
                "low"    { "Yellow" }
                default  { "Green" }
            }
        )
        Write-Host "Risk state:      $($riskyUser.RiskState)"
        Write-Host "Risk detail:     $($riskyUser.RiskDetail)"
        Write-Host "Last updated:    $($riskyUser.RiskLastUpdatedDateTime)"
    } else {
        Write-Host "Risk level:      None detected" -ForegroundColor Green
    }
}

# ============================================================
# FUNCTION 4 — Check if user has privileged role assignments
# ============================================================
function Get-UserPrivilegeLevel {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName
    )

    Write-Host "`n=== Privilege Assessment: $UserPrincipalName ===" -ForegroundColor Cyan

    $user = Get-MgUser -Filter "userPrincipalName eq '$UserPrincipalName'" `
            -Property "id,displayName"

    # High-risk roles — if user has any of these, treat as critical
    $criticalRoles = @(
        "Global Administrator",
        "Privileged Role Administrator",
        "Security Administrator",
        "Exchange Administrator",
        "SharePoint Administrator",
        "Conditional Access Administrator",
        "Authentication Administrator",
        "User Administrator"
    )

    $memberOf = Get-MgUserMemberOf -UserId $user.Id -All

    $directoryRoles = $memberOf | Where-Object {
        $_.AdditionalProperties["@odata.type"] -eq "#microsoft.graph.directoryRole"
    }

    if ($directoryRoles) {
        Write-Host "Directory roles assigned:" -ForegroundColor Yellow
        foreach ($role in $directoryRoles) {
            $roleName = $role.AdditionalProperties["displayName"]
            $isCritical = $criticalRoles -contains $roleName
            Write-Host "  $roleName" -ForegroundColor $(if ($isCritical) { "Red" } else { "Yellow" })
            if ($isCritical) {
                Write-Host "  ⚠ CRITICAL ROLE — escalate immediately if account is compromised" -ForegroundColor Red
            }
        }
    } else {
        Write-Host "No directory roles assigned — standard user account." -ForegroundColor Green
    }
}

# ============================================================
# FUNCTION 5 — Full triage summary (runs all functions)
# ============================================================
function Invoke-UserTriage {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [int]$SignInDaysBack = 7,
        [int]$AuditHoursBack = 48
    )

    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Write-Host "`n================================================" -ForegroundColor Cyan
    Write-Host "  IAM FIRST RESPONDER TRIAGE REPORT" -ForegroundColor Cyan
    Write-Host "  Target:    $UserPrincipalName" -ForegroundColor Cyan
    Write-Host "  Initiated: $timestamp" -ForegroundColor Cyan
    Write-Host "================================================`n" -ForegroundColor Cyan

    Get-UserRiskState -UserPrincipalName $UserPrincipalName
    Get-UserPrivilegeLevel -UserPrincipalName $UserPrincipalName
    Get-UserSignInSummary -UserPrincipalName $UserPrincipalName -DaysBack $SignInDaysBack
    Get-UserAuditTrail -UserPrincipalName $UserPrincipalName -HoursBack $AuditHoursBack

    Write-Host "`n================================================" -ForegroundColor Cyan
    Write-Host "  END OF TRIAGE REPORT" -ForegroundColor Cyan
    Write-Host "  Document findings before taking containment action" -ForegroundColor Yellow
    Write-Host "================================================`n" -ForegroundColor Cyan
}

Write-Host "IR Functions loaded. Available commands:" -ForegroundColor Green
Write-Host "  Invoke-UserTriage -UserPrincipalName <UPN>"
Write-Host "  Get-UserSignInSummary -UserPrincipalName <UPN> -DaysBack 7"
Write-Host "  Get-UserAuditTrail -UserPrincipalName <UPN> -HoursBack 48"
Write-Host "  Get-UserRiskState -UserPrincipalName <UPN>"
Write-Host "  Get-UserPrivilegeLevel -UserPrincipalName <UPN>"
```

---

### Step 3 — Test the Toolkit Against Your Own Account

```powershell
# Source the functions
. .\IR-Functions.ps1

# Run full triage against your own account
# This is safe — read only, no changes made
Invoke-UserTriage -UserPrincipalName "your.email@yourtenant.onmicrosoft.com"
```

Read every section of the output carefully. This is what you would be reading on a real incident.

---

### What You Learned
- How to connect to Graph with investigation-appropriate scopes
- Five reusable IR functions that cover the full triage picture
- The mental model: risk state → privilege level → sign-in history → audit trail

### Interview Talking Point After This Lab
*"My first action when an identity alert fires is running a structured triage. I have PowerShell functions built on Microsoft Graph that pull the four things I need in the first five minutes: current risk state from Identity Protection, privilege level of the affected account, sign-in history with IP and location, and the audit trail of any directory changes the account made. That structure means my first five minutes on an incident are consistent and nothing gets missed."*

---

## Lab 2 — Detect and Investigate Impossible Travel

### What You Will Learn
- How to detect impossible travel across the tenant — not just for one user
- How to calculate whether travel between two locations is physically possible
- How to distinguish legitimate VPN/proxy use from genuine impossible travel
- How to contain and document the finding

### Why This Matters for BoC
Impossible travel is one of the most common Identity Protection detections in financial services. A BoC panel may ask you to walk through an impossible travel investigation end to end.

### Reference Learning
- 📖 Microsoft Learn: [Identity Protection risk detections](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-risks)
- 📖 Microsoft Learn: [Investigate risk — Microsoft Entra ID Protection](https://learn.microsoft.com/en-us/entra/id-protection/howto-identity-protection-investigate-risk)

---

### Step 1 — Detect All Risky Sign-Ins Across the Tenant

```powershell
# Connect first
. .\Connect-IRSession.ps1
. .\IR-Functions.ps1

# Pull all medium and high risk sign-ins from the last 24 hours
function Get-TenantRiskySignIns {
    param(
        [int]$HoursBack = 24,
        [ValidateSet("low","medium","high")]
        [string]$MinRiskLevel = "medium"
    )

    Write-Host "`n=== Tenant-Wide Risky Sign-In Scan ===" -ForegroundColor Cyan
    Write-Host "Window: Last $HoursBack hours | Min risk: $MinRiskLevel`n"

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    # Map risk level to filter values
    $riskFilter = switch ($MinRiskLevel) {
        "high"   { "riskLevelDuringSignIn eq 'high'" }
        "medium" { "riskLevelDuringSignIn eq 'high' or riskLevelDuringSignIn eq 'medium'" }
        "low"    { "riskLevelDuringSignIn eq 'high' or riskLevelDuringSignIn eq 'medium' or riskLevelDuringSignIn eq 'low'" }
    }

    $riskySignIns = Get-MgAuditLogSignIn `
        -Filter "createdDateTime ge $startDate and ($riskFilter)" `
        -All `
        -Property "createdDateTime,userPrincipalName,ipAddress,location,
                   riskLevelDuringSignIn,riskEventTypes,status,appDisplayName"

    if (-not $riskySignIns) {
        Write-Host "No risky sign-ins detected in this window." -ForegroundColor Green
        return
    }

    Write-Host "Risky sign-ins found: $($riskySignIns.Count)" -ForegroundColor Red

    # Group by user to identify repeat offenders
    $byUser = $riskySignIns | Group-Object UserPrincipalName | Sort-Object Count -Descending

    Write-Host "`n--- Affected Users (sorted by incident count) ---"
    foreach ($group in $byUser) {
        $events = $group.Group
        $riskTypes = $events | ForEach-Object { $_.RiskEventTypes } |
                     Select-Object -Unique | Where-Object { $_ }

        Write-Host "`nUser: $($group.Name)" -ForegroundColor Yellow
        Write-Host "  Risky sign-ins: $($group.Count)"
        Write-Host "  Risk types: $($riskTypes -join ', ')"
        Write-Host "  IPs seen: $(($events.IpAddress | Select-Object -Unique) -join ', ')"
        Write-Host "  Countries: $(($events.Location.CountryOrRegion | Select-Object -Unique) -join ', ')"
    }

    return $riskySignIns
}

$riskySignIns = Get-TenantRiskySignIns -HoursBack 24 -MinRiskLevel "medium"
```

---

### Step 2 — Detect Impossible Travel Specifically

```powershell
function Find-ImpossibleTravel {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [int]$HoursBack = 24,
        # Minimum km/h that would be required between sign-ins to flag as impossible
        # Commercial aircraft max ~900 km/h — anything requiring >900 km/h is impossible
        [int]$MaxRealisticSpeedKmh = 900
    )

    Write-Host "`n=== Impossible Travel Analysis: $UserPrincipalName ===" -ForegroundColor Cyan

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    $signIns = Get-MgAuditLogSignIn `
        -Filter "userPrincipalName eq '$UserPrincipalName' and createdDateTime ge $startDate and status/errorCode eq 0" `
        -All `
        -Property "createdDateTime,ipAddress,location" |
        Sort-Object CreatedDateTime

    if ($signIns.Count -lt 2) {
        Write-Host "Fewer than 2 successful sign-ins in window — cannot calculate travel." -ForegroundColor Yellow
        return
    }

    Write-Host "Analysing $($signIns.Count) successful sign-ins for travel anomalies...`n"

    # Helper function — Haversine formula to calculate distance between coordinates
    function Get-DistanceKm {
        param($lat1, $lon1, $lat2, $lon2)
        $R = 6371  # Earth radius in km
        $dLat = [Math]::PI / 180 * ($lat2 - $lat1)
        $dLon = [Math]::PI / 180 * ($lon2 - $lon1)
        $a = [Math]::Sin($dLat/2) * [Math]::Sin($dLat/2) +
             [Math]::Cos([Math]::PI / 180 * $lat1) *
             [Math]::Cos([Math]::PI / 180 * $lat2) *
             [Math]::Sin($dLon/2) * [Math]::Sin($dLon/2)
        $c = 2 * [Math]::Atan2([Math]::Sqrt($a), [Math]::Sqrt(1 - $a))
        return [Math]::Round($R * $c, 0)
    }

    $anomaliesFound = $false

    for ($i = 1; $i -lt $signIns.Count; $i++) {
        $prev = $signIns[$i - 1]
        $curr = $signIns[$i]

        # Skip if no geo coordinates available
        if (-not $prev.Location.GeoCoordinates -or -not $curr.Location.GeoCoordinates) {
            continue
        }

        $prevLat = $prev.Location.GeoCoordinates.Latitude
        $prevLon = $prev.Location.GeoCoordinates.Longitude
        $currLat = $curr.Location.GeoCoordinates.Latitude
        $currLon = $curr.Location.GeoCoordinates.Longitude

        # Calculate distance
        $distanceKm = Get-DistanceKm $prevLat $prevLon $currLat $currLon

        # Calculate time difference in hours
        $timeDiff = ($curr.CreatedDateTime - $prev.CreatedDateTime).TotalHours

        # Skip if same location or negligible distance
        if ($distanceKm -lt 50) { continue }

        # Calculate required speed
        if ($timeDiff -gt 0) {
            $requiredSpeedKmh = [Math]::Round($distanceKm / $timeDiff, 0)
        } else {
            $requiredSpeedKmh = 999999  # Same timestamp different location = instant travel
        }

        $isImpossible = $requiredSpeedKmh -gt $MaxRealisticSpeedKmh

        if ($isImpossible) {
            $anomaliesFound = $true
            Write-Host "⚠ IMPOSSIBLE TRAVEL DETECTED" -ForegroundColor Red
            Write-Host "  From: $($prev.Location.City), $($prev.Location.CountryOrRegion) ($($prev.IpAddress))"
            Write-Host "  To:   $($curr.Location.City), $($curr.Location.CountryOrRegion) ($($curr.IpAddress))"
            Write-Host "  Distance:      $distanceKm km"
            Write-Host "  Time between:  $([Math]::Round($timeDiff * 60, 0)) minutes"
            Write-Host "  Required speed: $requiredSpeedKmh km/h (max realistic: $MaxRealisticSpeedKmh km/h)"
            Write-Host "  Signed-in at:  $($curr.CreatedDateTime)"
            Write-Host ""
        }
    }

    if (-not $anomaliesFound) {
        Write-Host "No impossible travel detected in this window." -ForegroundColor Green
        Write-Host "Note: VPN use or proxy servers can mask real location — verify with user if suspicious." -ForegroundColor Yellow
    }
}

# Run against a specific user
Find-ImpossibleTravel -UserPrincipalName "user@yourtenant.onmicrosoft.com" -HoursBack 48
```

---

### Step 3 — Investigate and Document the Finding

```powershell
function Invoke-ImpossibleTravelResponse {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [Parameter(Mandatory)]
        [string]$SuspiciousIP,
        [string]$IncidentTicketNumber = "INC-PENDING"
    )

    Write-Host "`n=== Impossible Travel — Structured Response ===" -ForegroundColor Cyan
    Write-Host "Ticket: $IncidentTicketNumber`n"

    # Step 1 — Full triage
    Invoke-UserTriage -UserPrincipalName $UserPrincipalName

    # Step 2 — Check if suspicious IP appears on other accounts
    Write-Host "`n=== Checking if suspicious IP hit other accounts ===" -ForegroundColor Yellow
    $startDate = (Get-Date).AddDays(-3).ToString("yyyy-MM-ddTHH:mm:ssZ")

    $otherSignIns = Get-MgAuditLogSignIn `
        -Filter "ipAddress eq '$SuspiciousIP' and createdDateTime ge $startDate" `
        -All `
        -Property "userPrincipalName,createdDateTime,status"

    $otherUsers = $otherSignIns |
        Where-Object { $_.UserPrincipalName -ne $UserPrincipalName } |
        Select-Object -ExpandProperty UserPrincipalName -Unique

    if ($otherUsers) {
        Write-Host "⚠ SAME IP ALSO USED BY:" -ForegroundColor Red
        $otherUsers | ForEach-Object { Write-Host "  $_" -ForegroundColor Red }
        Write-Host "This may indicate a shared threat actor — expand investigation scope."
    } else {
        Write-Host "IP appears isolated to this user — no lateral spread detected." -ForegroundColor Green
    }

    # Step 3 — Generate response recommendation
    $user = Get-MgUser -Filter "userPrincipalName eq '$UserPrincipalName'" `
            -Property "id,accountEnabled"
    $riskyUser = Get-MgRiskyUser -Filter "userPrincipalName eq '$UserPrincipalName'"

    Write-Host "`n=== Response Recommendation ===" -ForegroundColor Yellow

    $riskLevel = $riskyUser.RiskLevel

    switch ($riskLevel) {
        "high" {
            Write-Host "RISK: HIGH — Recommended action: Revoke sessions + Confirm compromised" -ForegroundColor Red
            Write-Host "Commands to run after verification:"
            Write-Host '  Invoke-MgGraphRequest -Method POST -Uri "https://graph.microsoft.com/v1.0/users/<userId>/revokeSignInSessions"'
            Write-Host '  Update-MgRiskyUser -UserId "<userId>" -RiskLevel "high" -IsProcessing $false'
        }
        "medium" {
            Write-Host "RISK: MEDIUM — Recommended action: Revoke sessions + contact user to verify" -ForegroundColor Yellow
            Write-Host "Do not disable account until user confirms they do not recognise the sign-in."
        }
        default {
            Write-Host "RISK: LOW/NONE — Verify with user. May be VPN or proxy." -ForegroundColor Green
            Write-Host "Check: is this user known to use a VPN that exits in unexpected countries?"
        }
    }
}

Invoke-ImpossibleTravelResponse `
    -UserPrincipalName "user@yourtenant.onmicrosoft.com" `
    -SuspiciousIP "1.2.3.4" `
    -IncidentTicketNumber "INC-2026-001"
```

---

### What You Learned
- How to scan the entire tenant for risky sign-ins in a single query
- How to implement the Haversine formula in PowerShell to calculate whether travel is physically possible
- How to check if a suspicious IP is being used across multiple accounts (lateral spread)
- How to generate a risk-proportionate response recommendation

### Interview Talking Point After This Lab
*"When an impossible travel alert fires I do not just look at the flagged user — I also check if the suspicious IP appeared on any other accounts in the same window. A single impossible travel event is often a compromised credential. The same IP hitting five accounts in the same hour is a coordinated attack. The scope of the investigation changes completely based on that second check."*

---

## Lab 3 — Detect and Respond to MFA Fatigue Attacks

### What You Will Learn
- How to detect MFA fatigue patterns — high push notification volume in a short window
- How to identify if a fatigue attack succeeded — the moment a user approved
- How to respond — revoke sessions, require re-registration, harden MFA method

### Why This Matters for BoC
MFA fatigue is one of the most common attack vectors against financial institutions right now. Uber, Microsoft, Cisco — all compromised via MFA fatigue in recent years. BoC will expect you to know this cold.

### Reference Learning
- 📖 Microsoft Learn: [Protect against MFA fatigue](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-mfa-additional-context)
- 🎥 YouTube: Search `"MFA fatigue attack Entra ID" 2025` — Microsoft Security YouTube channel

---

### Step 1 — Detect MFA Fatigue Patterns

```powershell
function Find-MfaFatigueAttack {
    param(
        [int]$HoursBack = 24,
        # How many MFA failures in a window before flagging
        [int]$FailureThreshold = 10,
        # Time window in minutes to count failures within
        [int]$WindowMinutes = 30
    )

    Write-Host "`n=== MFA Fatigue Attack Detection ===" -ForegroundColor Cyan
    Write-Host "Window: Last $HoursBack hours"
    Write-Host "Threshold: $FailureThreshold MFA failures within $WindowMinutes minutes`n"

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    # MFA failure error codes
    # 500121 = Authentication failed during strong authentication request
    # 50074  = Strong Authentication required
    # 50076  = Strong Authentication required (sign-in interrupted)
    # 50079  = User needs MFA registration
    $mfaFailureCodes = @("500121", "50074", "50076")

    $mfaFailures = Get-MgAuditLogSignIn `
        -Filter "createdDateTime ge $startDate and (status/errorCode eq 500121 or status/errorCode eq 50074 or status/errorCode eq 50076)" `
        -All `
        -Property "createdDateTime,userPrincipalName,ipAddress,status,authenticationDetails"

    if (-not $mfaFailures) {
        Write-Host "No MFA failure patterns detected." -ForegroundColor Green
        return
    }

    Write-Host "Total MFA failures in window: $($mfaFailures.Count)"

    # Group by user and analyse patterns
    $byUser = $mfaFailures | Group-Object UserPrincipalName

    $fatigueTargets = @()

    foreach ($group in $byUser) {
        $events = $group.Group | Sort-Object CreatedDateTime

        # Sliding window analysis
        for ($i = 0; $i -lt $events.Count; $i++) {
            $windowStart = $events[$i].CreatedDateTime
            $windowEnd = $windowStart.AddMinutes($WindowMinutes)

            $inWindow = $events | Where-Object {
                $_.CreatedDateTime -ge $windowStart -and
                $_.CreatedDateTime -le $windowEnd
            }

            if ($inWindow.Count -ge $FailureThreshold) {
                $fatigueTargets += [PSCustomObject]@{
                    UserPrincipalName = $group.Name
                    FailuresInWindow  = $inWindow.Count
                    WindowStart       = $windowStart
                    WindowEnd         = $windowEnd
                    UniqueIPs         = ($inWindow.IpAddress | Select-Object -Unique) -join ", "
                }
                break  # Found fatigue pattern for this user — move to next user
            }
        }
    }

    if ($fatigueTargets) {
        Write-Host "`n⚠ MFA FATIGUE TARGETS DETECTED:" -ForegroundColor Red
        $fatigueTargets | ForEach-Object {
            Write-Host "`nUser: $($_.UserPrincipalName)" -ForegroundColor Red
            Write-Host "  MFA failures in $WindowMinutes min window: $($_.FailuresInWindow)"
            Write-Host "  Window: $($_.WindowStart) to $($_.WindowEnd)"
            Write-Host "  Attacker IPs: $($_.UniqueIPs)"
        }
    } else {
        Write-Host "`nNo fatigue patterns detected above threshold." -ForegroundColor Green
    }

    return $fatigueTargets
}

$fatigueTargets = Find-MfaFatigueAttack -HoursBack 24 -FailureThreshold 10 -WindowMinutes 30
```

---

### Step 2 — Check If the Attack Succeeded

```powershell
function Test-MfaFatigueSuccess {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [Parameter(Mandatory)]
        [datetime]$AttackWindowStart,
        [Parameter(Mandatory)]
        [datetime]$AttackWindowEnd
    )

    Write-Host "`n=== Checking if MFA Fatigue Attack Succeeded ===" -ForegroundColor Cyan
    Write-Host "User: $UserPrincipalName"
    Write-Host "Attack window: $AttackWindowStart to $AttackWindowEnd`n"

    $startFilter = $AttackWindowStart.ToString("yyyy-MM-ddTHH:mm:ssZ")
    $endFilter = $AttackWindowEnd.AddHours(1).ToString("yyyy-MM-ddTHH:mm:ssZ")

    # Look for a successful sign-in during or just after the attack window
    $successfulSignIns = Get-MgAuditLogSignIn `
        -Filter "userPrincipalName eq '$UserPrincipalName' and createdDateTime ge $startFilter and createdDateTime le $endFilter and status/errorCode eq 0" `
        -All `
        -Property "createdDateTime,ipAddress,location,appDisplayName,authenticationDetails,deviceDetail"

    if ($successfulSignIns) {
        Write-Host "⚠ ATTACK MAY HAVE SUCCEEDED" -ForegroundColor Red
        Write-Host "Successful sign-in(s) found during/after attack window:`n"

        $successfulSignIns | ForEach-Object {
            Write-Host "  Time:        $($_.CreatedDateTime)" -ForegroundColor Red
            Write-Host "  IP:          $($_.IpAddress)"
            Write-Host "  Location:    $($_.Location.City), $($_.Location.CountryOrRegion)"
            Write-Host "  Application: $($_.AppDisplayName)"
            Write-Host "  Device:      $($_.DeviceDetail.DisplayName) ($($_.DeviceDetail.OperatingSystem))"
            Write-Host ""
        }

        Write-Host "ACTION REQUIRED: Investigate what the account accessed during this session." -ForegroundColor Red
        Write-Host "Run: Get-UserAuditTrail -UserPrincipalName '$UserPrincipalName' -HoursBack 12"

    } else {
        Write-Host "No successful sign-ins found during or after attack window." -ForegroundColor Green
        Write-Host "Attack appears to have been blocked — user did not approve MFA." -ForegroundColor Green
        Write-Host "Recommendation: Contact user to confirm, then harden MFA method to number matching or FIDO2." -ForegroundColor Yellow
    }
}

# Example — test if attack on a specific user succeeded
# Test-MfaFatigueSuccess `
#     -UserPrincipalName "user@yourtenant.onmicrosoft.com" `
#     -AttackWindowStart (Get-Date).AddHours(-5) `
#     -AttackWindowEnd (Get-Date).AddHours(-4)
```

---

### Step 3 — Contain and Harden

```powershell
function Invoke-MfaFatigueContainment {
    param(
        [Parameter(Mandatory)]
        [string]$UserId,
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [string]$IncidentTicket = "INC-PENDING",
        [switch]$AttackSucceeded
    )

    Write-Host "`n=== MFA Fatigue Containment ===" -ForegroundColor Red
    Write-Host "User: $UserPrincipalName | Ticket: $IncidentTicket`n"

    # Step 1 — Revoke sessions (always, regardless of whether attack succeeded)
    Write-Host "Step 1: Revoking all active sessions..." -ForegroundColor Yellow
    Invoke-MgGraphRequest -Method POST `
        -Uri "https://graph.microsoft.com/v1.0/users/$UserId/revokeSignInSessions"
    Write-Host "Sessions revoked — active tokens invalidated for CAE-capable apps." -ForegroundColor Green

    # Step 2 — If attack succeeded, disable account pending investigation
    if ($AttackSucceeded) {
        Write-Host "`nStep 2: Attack succeeded — disabling account pending investigation..." -ForegroundColor Red
        Update-MgUser -UserId $UserId -AccountEnabled $false
        Write-Host "Account disabled." -ForegroundColor Red
        Write-Host "Remember: use a Temporary Access Pass to restore access after remediation."
    } else {
        Write-Host "`nStep 2: Attack blocked — account remains enabled." -ForegroundColor Green
        Write-Host "User should be contacted to verify they recognise the MFA prompts."
    }

    # Step 3 — Review MFA methods registered on account
    Write-Host "`nStep 3: Reviewing registered MFA methods..." -ForegroundColor Yellow
    $authMethods = Get-MgUserAuthenticationMethod -UserId $UserId

    Write-Host "Current authentication methods:"
    foreach ($method in $authMethods) {
        $methodType = $method.AdditionalProperties["@odata.type"]
        Write-Host "  $methodType" -ForegroundColor $(
            if ($methodType -like "*microsoftAuthenticator*") { "Yellow" }
            elseif ($methodType -like "*fido2*") { "Green" }
            elseif ($methodType -like "*phone*") { "Red" }
            else { "White" }
        )
    }

    Write-Host "`nRecommendation after investigation is complete:"
    Write-Host "  1. Issue a Temporary Access Pass for re-authentication"
    Write-Host "  2. Remove push notification MFA method"
    Write-Host "  3. Register FIDO2 security key or Windows Hello for Business"
    Write-Host "  4. Push notification is susceptible to fatigue — FIDO2 is not"

    # Step 4 — Elevate user risk to force remediation on next sign-in
    Write-Host "`nStep 4: Confirming user as compromised to force remediation flow..."
    Invoke-MgGraphRequest -Method POST `
        -Uri "https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/confirmCompromised" `
        -Body (@{ userIds = @($UserId) } | ConvertTo-Json) `
        -ContentType "application/json"
    Write-Host "User risk set to High — password reset required on next sign-in." -ForegroundColor Green
}
```

---

### What You Learned
- MFA fatigue detection using sliding window analysis on sign-in error codes
- How to determine if the attack succeeded by correlating failures with successful sign-ins
- The escalating containment response — session revocation first, account disable only if attack succeeded
- Why push notification MFA is vulnerable and FIDO2 is the correct hardening action

### Interview Talking Point After This Lab
*"MFA fatigue detection requires a sliding window analysis — you are not just counting total failures for a user, you are looking for a burst of failures within a narrow time window because that is the attack pattern. The second question after detecting the fatigue is whether it succeeded — I correlate the failure burst with successful sign-ins in the same window. If there is a successful sign-in during or just after the attack, the user approved under pressure and I treat it as a confirmed compromise. If there is no successful sign-in, I contact the user to confirm, then use the incident as the trigger to move them from push notification to FIDO2 — because push notification is inherently susceptible to fatigue and FIDO2 is not."*

---

## Lab 4 — Hunt for Credential Stuffing Across the Tenant

### What You Will Learn
- How to identify credential stuffing — high failure volume from one IP across many accounts
- How to distinguish credential stuffing from a single compromised account
- How to extract the list of targeted accounts for follow-up investigation

### Why This Matters for BoC
Credential stuffing is the most common initial access technique against financial institutions. Detecting it quickly — before any account is successfully accessed — is the highest-value IAM first responder activity.

---

### Step 1 — Detect Cross-Account Attacks From Single IP

```powershell
function Find-CredentialStuffing {
    param(
        [int]$HoursBack = 24,
        # IP hitting this many unique accounts is likely stuffing
        [int]$UniqueAccountThreshold = 5,
        # Minimum failures per IP to include in analysis
        [int]$MinFailuresPerIP = 20
    )

    Write-Host "`n=== Credential Stuffing Detection ===" -ForegroundColor Cyan
    Write-Host "Window: Last $HoursBack hours"
    Write-Host "Thresholds: $MinFailuresPerIP+ failures, $UniqueAccountThreshold+ unique accounts per IP`n"

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    # Get all failed sign-ins — error code 50126 = invalid credentials
    # 50053 = account locked, 50055 = expired password, 50056 = no password
    $failedSignIns = Get-MgAuditLogSignIn `
        -Filter "createdDateTime ge $startDate and status/errorCode eq 50126" `
        -All `
        -Property "createdDateTime,userPrincipalName,ipAddress,location,status"

    if (-not $failedSignIns) {
        Write-Host "No invalid credential failures detected." -ForegroundColor Green
        return
    }

    Write-Host "Total invalid credential failures: $($failedSignIns.Count)`n"

    # Group by IP address
    $byIP = $failedSignIns | Group-Object IpAddress |
            Where-Object { $_.Count -ge $MinFailuresPerIP } |
            Sort-Object Count -Descending

    if (-not $byIP) {
        Write-Host "No IPs exceeded failure threshold — no stuffing pattern detected." -ForegroundColor Green
        return
    }

    $stuffingIPs = @()

    foreach ($ipGroup in $byIP) {
        $events = $ipGroup.Group
        $uniqueAccounts = $events | Select-Object -ExpandProperty UserPrincipalName -Unique
        $successfulFromIP = Get-MgAuditLogSignIn `
            -Filter "ipAddress eq '$($ipGroup.Name)' and createdDateTime ge $startDate and status/errorCode eq 0" `
            -All `
            -Property "userPrincipalName,createdDateTime"

        if ($uniqueAccounts.Count -ge $UniqueAccountThreshold) {
            $stuffingIPs += [PSCustomObject]@{
                IP              = $ipGroup.Name
                TotalFailures   = $ipGroup.Count
                UniqueAccounts  = $uniqueAccounts.Count
                SuccessfulLogins = $successfulFromIP.Count
                TargetedAccounts = $uniqueAccounts
                Location        = ($events | Select-Object -First 1).Location.CountryOrRegion
            }

            Write-Host "⚠ CREDENTIAL STUFFING DETECTED" -ForegroundColor Red
            Write-Host "  IP:               $($ipGroup.Name) ($($events[0].Location.CountryOrRegion))"
            Write-Host "  Total failures:   $($ipGroup.Count)"
            Write-Host "  Unique accounts:  $($uniqueAccounts.Count)"
            Write-Host "  Successful logins from this IP: $($successfulFromIP.Count)" -ForegroundColor $(
                if ($successfulFromIP.Count -gt 0) { "Red" } else { "Green" }
            )

            if ($successfulFromIP.Count -gt 0) {
                Write-Host "  ⚠ COMPROMISED ACCOUNTS FROM THIS IP:" -ForegroundColor Red
                $successfulFromIP | ForEach-Object {
                    Write-Host "    $($_.UserPrincipalName) at $($_.CreatedDateTime)" -ForegroundColor Red
                }
            }

            Write-Host "  Targeted accounts (first 10):"
            $uniqueAccounts | Select-Object -First 10 |
                ForEach-Object { Write-Host "    $_" }
            Write-Host ""
        }
    }

    return $stuffingIPs
}

$stuffingIPs = Find-CredentialStuffing -HoursBack 24 -UniqueAccountThreshold 5 -MinFailuresPerIP 20
```

---

### Step 2 — Triage Successfully Compromised Accounts

```powershell
function Invoke-StuffingCompromisedAccountTriage {
    param(
        [Parameter(Mandatory)]
        [string[]]$CompromisedUPNs,
        [string]$AttackerIP
    )

    Write-Host "`n=== Bulk Compromise Triage ===" -ForegroundColor Red
    Write-Host "Accounts to investigate: $($CompromisedUPNs.Count)`n"

    $report = @()

    foreach ($upn in $CompromisedUPNs) {
        Write-Host "--- Triaging: $upn ---" -ForegroundColor Yellow

        $user = Get-MgUser -Filter "userPrincipalName eq '$upn'" `
                -Property "id,displayName,accountEnabled,jobTitle,department"

        $riskyUser = Get-MgRiskyUser -Filter "userPrincipalName eq '$upn'"

        # Check if user has privileged roles
        $memberOf = Get-MgUserMemberOf -UserId $user.Id -All
        $hasAdminRoles = $memberOf | Where-Object {
            $_.AdditionalProperties["@odata.type"] -eq "#microsoft.graph.directoryRole"
        }

        $report += [PSCustomObject]@{
            UPN           = $upn
            DisplayName   = $user.DisplayName
            Department    = $user.Department
            AccountEnabled = $user.AccountEnabled
            RiskLevel     = $riskyUser.RiskLevel
            HasAdminRoles = ($hasAdminRoles.Count -gt 0)
            AdminRoles    = ($hasAdminRoles.AdditionalProperties.displayName -join ", ")
            Priority      = if ($hasAdminRoles.Count -gt 0) { "CRITICAL" }
                           elseif ($riskyUser.RiskLevel -eq "high") { "HIGH" }
                           else { "MEDIUM" }
        }
    }

    # Sort by priority
    $prioritised = $report | Sort-Object @{
        Expression = {
            switch ($_.Priority) {
                "CRITICAL" { 1 }
                "HIGH"     { 2 }
                "MEDIUM"   { 3 }
            }
        }
    }

    Write-Host "`n=== TRIAGE RESULTS (prioritised) ===" -ForegroundColor Cyan
    $prioritised | ForEach-Object {
        $colour = switch ($_.Priority) {
            "CRITICAL" { "Red" }
            "HIGH"     { "Yellow" }
            "MEDIUM"   { "White" }
        }
        Write-Host "[$($_.Priority)] $($_.UPN)" -ForegroundColor $colour
        if ($_.HasAdminRoles) {
            Write-Host "  Admin roles: $($_.AdminRoles)" -ForegroundColor Red
        }
    }

    return $prioritised
}
```

---

### What You Learned
- Credential stuffing signature: high failure count from one IP across many unique accounts
- How to determine if the stuffing succeeded by checking successful sign-ins from the same IP
- How to prioritise response — privileged accounts first, then high-risk, then standard users

### Interview Talking Point After This Lab
*"The key distinction between credential stuffing and a single compromised account is the IP-to-account ratio. One account failing many times from many IPs is a user who forgot their password. One IP failing against many unique accounts in a short window is credential stuffing — a known breach list being tested. When I detect stuffing, the first question is whether any attempt succeeded. I pull successful sign-ins from the same IP in the same window. If I find them, those accounts are compromised and I triage them by privilege level — admin accounts first."*

---

## Lab 5 — Detect App Registration Persistence Mechanisms

### What You Will Learn
- How attackers use App Registration credential additions as persistence
- How to audit all recent credential changes across every App Registration in the tenant
- How to identify orphaned or unowned App Registrations that are high-risk

### Why This Matters for BoC
Adding a credential to an App Registration is the most dangerous persistence mechanism in Entra ID. It survives account disables, password resets, and MFA changes. A BoC technical panel will expect you to know this.

---

### Step 1 — Detect Recent Credential Additions to App Registrations

```powershell
function Find-AppRegistrationCredentialChanges {
    param(
        [int]$HoursBack = 48
    )

    Write-Host "`n=== App Registration Credential Change Audit ===" -ForegroundColor Cyan
    Write-Host "Window: Last $HoursBack hours`n"

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    # Operations that indicate credential manipulation
    $credentialOperations = @(
        "Add service principal credentials",
        "Remove service principal credentials",
        "Update application – Certificates and secrets management",
        "Add application credentials",
        "Remove application credentials"
    )

    $filterString = ($credentialOperations |
        ForEach-Object { "activityDisplayName eq '$_'" }) -join " or "

    $auditEvents = Get-MgAuditLogDirectoryAudit `
        -Filter "activityDateTime ge $startDate and ($filterString)" `
        -All `
        -Property "activityDateTime,activityDisplayName,result,
                   initiatedBy,targetResources,additionalDetails"

    if (-not $auditEvents) {
        Write-Host "No credential changes detected in this window." -ForegroundColor Green
        return
    }

    Write-Host "Credential changes found: $($auditEvents.Count)" -ForegroundColor $(
        if ($auditEvents.Count -gt 5) { "Red" } else { "Yellow" }
    )

    foreach ($event in $auditEvents | Sort-Object ActivityDateTime -Descending) {
        $actor = $event.InitiatedBy.User.UserPrincipalName ??
                 $event.InitiatedBy.App.DisplayName ??
                 "Unknown"
        $target = $event.TargetResources[0].DisplayName

        # Flag if initiated by a service principal (not a human) — unusual
        $initiatedByApp = $null -ne $event.InitiatedBy.App.DisplayName
        $flag = if ($initiatedByApp) { "⚠ INITIATED BY APP — INVESTIGATE" } else { "" }

        Write-Host "`n$($event.ActivityDateTime) | $($event.ActivityDisplayName)"
        Write-Host "  Actor:  $actor $(if ($initiatedByApp) { '(SERVICE PRINCIPAL)' })" `
            -ForegroundColor $(if ($initiatedByApp) { "Red" } else { "White" })
        Write-Host "  Target: $target"
        Write-Host "  Result: $($event.Result)"
        if ($flag) { Write-Host "  $flag" -ForegroundColor Red }
    }
}

Find-AppRegistrationCredentialChanges -HoursBack 48
```

---

### Step 2 — Audit All App Registrations for Expiring or Suspicious Credentials

```powershell
function Get-AppRegistrationCredentialAudit {
    param(
        # Flag certs/secrets expiring within this many days
        [int]$ExpiryWarningDays = 90,
        # Flag apps with no owners (unaccountable)
        [switch]$FlagNoOwners,
        # Flag apps with credentials but no sign-in activity
        [switch]$FlagDormant
    )

    Write-Host "`n=== Tenant-Wide App Registration Credential Audit ===" -ForegroundColor Cyan

    $apps = Get-MgApplication -All `
            -Property "id,appId,displayName,createdDateTime,
                       keyCredentials,passwordCredentials,owners"

    Write-Host "Total App Registrations: $($apps.Count)`n"

    $findings = @()

    foreach ($app in $apps) {
        $issues = @()

        # Check for expiring secrets
        foreach ($secret in $app.PasswordCredentials) {
            $daysToExpiry = ($secret.EndDateTime - (Get-Date)).TotalDays
            if ($daysToExpiry -lt 0) {
                $issues += "EXPIRED secret: $($secret.DisplayName) (expired $([Math]::Abs([int]$daysToExpiry)) days ago)"
            } elseif ($daysToExpiry -lt $ExpiryWarningDays) {
                $issues += "EXPIRING secret: $($secret.DisplayName) (expires in $([int]$daysToExpiry) days)"
            }
        }

        # Check for expiring certificates
        foreach ($cert in $app.KeyCredentials) {
            $daysToExpiry = ($cert.EndDateTime - (Get-Date)).TotalDays
            if ($daysToExpiry -lt 0) {
                $issues += "EXPIRED cert: $($cert.DisplayName)"
            } elseif ($daysToExpiry -lt $ExpiryWarningDays) {
                $issues += "EXPIRING cert: $($cert.DisplayName) (expires in $([int]$daysToExpiry) days)"
            }
        }

        # Check for no owners
        if ($FlagNoOwners) {
            $owners = Get-MgApplicationOwner -ApplicationId $app.Id
            if (-not $owners) {
                $issues += "NO OWNERS — unaccountable application"
            }
        }

        # Check for multiple secrets (more than 2 is unusual)
        if ($app.PasswordCredentials.Count -gt 2) {
            $issues += "EXCESSIVE SECRETS: $($app.PasswordCredentials.Count) client secrets — review needed"
        }

        if ($issues) {
            $findings += [PSCustomObject]@{
                AppName  = $app.DisplayName
                AppId    = $app.AppId
                Created  = $app.CreatedDateTime
                Issues   = $issues
            }
        }
    }

    if ($findings) {
        Write-Host "Apps with findings: $($findings.Count)" -ForegroundColor Yellow
        foreach ($finding in $findings) {
            Write-Host "`nApp: $($finding.AppName) ($($finding.AppId))" -ForegroundColor Yellow
            $finding.Issues | ForEach-Object {
                $colour = if ($_ -like "EXPIRED*" -or $_ -like "NO OWNERS*" -or $_ -like "EXCESSIVE*") {
                    "Red"
                } else { "Yellow" }
                Write-Host "  → $_" -ForegroundColor $colour
            }
        }
    } else {
        Write-Host "No credential issues found." -ForegroundColor Green
    }

    return $findings
}

Get-AppRegistrationCredentialAudit -ExpiryWarningDays 90 -FlagNoOwners -FlagDormant
```

---

### What You Learned
- How attackers use App Registration credential additions as persistence — it survives account disable and password reset
- How to detect credential changes across the entire tenant in a single query
- How to flag unowned, over-credentialed, and expiring App Registrations at scale

### Interview Talking Point After This Lab
*"App Registration credential addition is the persistence mechanism most IAM engineers miss. When a user account is compromised and we disable it, revoke sessions, and reset the password — all of that is undone if the attacker added a client secret to an App Registration the account owned. The attacker now authenticates as that service principal indefinitely. So my IR checklist always includes checking whether any App Registration credentials were added during the suspicious window — and specifically whether the changes were initiated by a service principal rather than a human, because that is the signal that automation was used to establish persistence."*

---

## Lab 6 — Investigate Suspicious PIM Activations

### What You Will Learn
- How to query the PIM audit log for all privileged role activations
- How to flag activations outside business hours or from unexpected locations
- How to correlate PIM activations with subsequent audit events

### Why This Matters for BoC
PIM protects the most sensitive roles. A compromised account that activates Global Admin has tenant-wide blast radius. Detecting and responding to suspicious PIM activations is a critical IAM first responder skill.

---

### Step 1 — Audit PIM Activations

```powershell
function Get-SuspiciousPimActivations {
    param(
        [int]$HoursBack = 48,
        # Business hours window — activations outside this are flagged
        [int]$BusinessHoursStart = 7,   # 7am
        [int]$BusinessHoursEnd = 19,    # 7pm
        # Roles considered critical — activation always investigated
        [string[]]$CriticalRoles = @(
            "Global Administrator",
            "Privileged Role Administrator",
            "Security Administrator",
            "Conditional Access Administrator",
            "Authentication Administrator",
            "User Administrator"
        )
    )

    Write-Host "`n=== Suspicious PIM Activation Audit ===" -ForegroundColor Cyan
    Write-Host "Window: Last $HoursBack hours`n"

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    # PIM activation events in the audit log
    $pimActivations = Get-MgAuditLogDirectoryAudit `
        -Filter "activityDateTime ge $startDate and (activityDisplayName eq 'Add member to role completed (PIM activation)' or activityDisplayName eq 'Add eligible member to role')" `
        -All `
        -Property "activityDateTime,activityDisplayName,result,
                   initiatedBy,targetResources"

    if (-not $pimActivations) {
        Write-Host "No PIM activations in this window." -ForegroundColor Green
        return
    }

    Write-Host "Total PIM activations: $($pimActivations.Count)`n"

    $findings = @()

    foreach ($activation in $pimActivations) {
        $actor = $activation.InitiatedBy.User.UserPrincipalName
        $role = $activation.TargetResources[0].DisplayName
        $activationTime = $activation.ActivityDateTime
        $hourOfDay = $activationTime.ToLocalTime().Hour

        $issues = @()

        # Flag critical roles
        if ($CriticalRoles -contains $role) {
            $issues += "CRITICAL ROLE: $role"
        }

        # Flag outside business hours
        if ($hourOfDay -lt $BusinessHoursStart -or $hourOfDay -ge $BusinessHoursEnd) {
            $issues += "OUTSIDE BUSINESS HOURS: $($activationTime.ToLocalTime().ToString('HH:mm')) local"
        }

        # Flag weekend activations
        if ($activationTime.DayOfWeek -in @("Saturday","Sunday")) {
            $issues += "WEEKEND ACTIVATION: $($activationTime.DayOfWeek)"
        }

        if ($issues) {
            $findings += [PSCustomObject]@{
                Time   = $activationTime
                Actor  = $actor
                Role   = $role
                Issues = $issues
            }

            Write-Host "⚠ FLAGGED ACTIVATION" -ForegroundColor Red
            Write-Host "  Time:   $activationTime"
            Write-Host "  Actor:  $actor"
            Write-Host "  Role:   $role"
            $issues | ForEach-Object { Write-Host "  Flag:   $_" -ForegroundColor Red }
            Write-Host ""
        } else {
            Write-Host "✓ $activationTime | $actor | $role" -ForegroundColor Green
        }
    }

    return $findings
}

$suspiciousActivations = Get-SuspiciousPimActivations -HoursBack 48
```

---

### Step 2 — Correlate PIM Activation With What the Account Did

```powershell
function Get-PostActivationAuditTrail {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [Parameter(Mandatory)]
        [datetime]$ActivationTime,
        # Look at audit events for this many hours after activation
        [int]$WindowHours = 4
    )

    Write-Host "`n=== Post-PIM-Activation Audit Trail ===" -ForegroundColor Cyan
    Write-Host "User: $UserPrincipalName"
    Write-Host "Activation at: $ActivationTime"
    Write-Host "Investigating next $WindowHours hours`n"

    $startFilter = $ActivationTime.ToString("yyyy-MM-ddTHH:mm:ssZ")
    $endFilter = $ActivationTime.AddHours($WindowHours).ToString("yyyy-MM-ddTHH:mm:ssZ")

    $auditEvents = Get-MgAuditLogDirectoryAudit `
        -Filter "initiatedBy/user/userPrincipalName eq '$UserPrincipalName' and activityDateTime ge $startFilter and activityDateTime le $endFilter" `
        -All `
        -Property "activityDateTime,activityDisplayName,result,targetResources"

    if (-not $auditEvents) {
        Write-Host "No audit events found after activation — role was activated but no changes were made." -ForegroundColor Green
        Write-Host "This could indicate reconnaissance or the session was interrupted." -ForegroundColor Yellow
        return
    }

    # Flag high-risk operations
    $highRiskOps = @(
        "Add member to role",
        "Update policy",
        "Add policy",
        "Delete policy",
        "Add service principal credentials",
        "Reset user password",
        "Update conditional access policy",
        "Create application",
        "Delete application",
        "Add member to group",
        "Delete user"
    )

    Write-Host "Audit events after PIM activation: $($auditEvents.Count)"
    Write-Host ""

    foreach ($event in $auditEvents | Sort-Object ActivityDateTime) {
        $isHighRisk = $highRiskOps | Where-Object { $event.ActivityDisplayName -like "*$_*" }
        $target = $event.TargetResources[0].DisplayName

        $colour = if ($isHighRisk) { "Red" } else { "White" }
        $flag = if ($isHighRisk) { " ⚠ HIGH RISK" } else { "" }

        Write-Host "$($event.ActivityDateTime) | $($event.ActivityDisplayName) | $target$flag" `
            -ForegroundColor $colour
    }
}

# Example usage after finding a suspicious activation
# Get-PostActivationAuditTrail `
#     -UserPrincipalName "admin@yourtenant.onmicrosoft.com" `
#     -ActivationTime (Get-Date).AddHours(-6)
```

---

### What You Learned
- How to query PIM audit logs and flag activations by time, day, and role criticality
- How to correlate a PIM activation with subsequent directory changes
- Why a PIM activation with no subsequent activity is still worth investigating — reconnaissance

### Interview Talking Point After This Lab
*"PIM activations outside business hours or on weekends are not automatically malicious — admins do legitimate out-of-hours work. But they are always worth a quick audit trail check. I correlate the activation time with the audit log for that user in the following four hours. A Global Admin activation at 2am where the account then added a new service principal credential, changed a CA policy, and created a new user is a different investigation than one where the activation happened and then nothing followed. The nothing is also interesting — it could mean reconnaissance, or it could mean the attacker was interrupted."*

---

## Lab 7 — Token Theft Detection and Containment

### What You Will Learn
- How to detect token replay attacks — access token used from a different IP than the original sign-in
- How Continuous Access Evaluation (CAE) is your fastest containment tool
- How to correlate Continuous Access Evaluation events in the sign-in logs

### Why This Matters for BoC
Token theft bypasses MFA entirely. The attacker never needs to authenticate — they just replay a stolen token. This is the attack technique that Adversary-in-the-Middle (AiTM) phishing enables. Detecting and revoking tokens is the containment.

---

### Step 1 — Detect Token Anomalies

```powershell
function Find-TokenAnomalies {
    param(
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [int]$HoursBack = 24
    )

    Write-Host "`n=== Token Anomaly Detection: $UserPrincipalName ===" -ForegroundColor Cyan
    Write-Host "Window: Last $HoursBack hours`n"

    $startDate = (Get-Date).AddHours(-$HoursBack).ToString("yyyy-MM-ddTHH:mm:ssZ")

    $signIns = Get-MgAuditLogSignIn `
        -Filter "userPrincipalName eq '$UserPrincipalName' and createdDateTime ge $startDate" `
        -All `
        -Property "createdDateTime,ipAddress,location,resourceDisplayName,
                   authenticationDetails,tokenIssuancePolicy,
                   riskEventTypes,riskLevelDuringSignIn,
                   conditionalAccessStatus,appliedConditionalAccessPolicies" |
        Sort-Object CreatedDateTime

    if (-not $signIns) {
        Write-Host "No sign-ins found in this window." -ForegroundColor Yellow
        return
    }

    # Look for anomalous_token risk event type
    $tokenRiskySignIns = $signIns | Where-Object {
        $_.RiskEventTypes -contains "anomalousToken" -or
        $_.RiskEventTypes -contains "tokenIssuerAnomaly" -or
        $_.RiskLevelDuringSignIn -in @("medium","high")
    }

    if ($tokenRiskySignIns) {
        Write-Host "⚠ TOKEN RISK EVENTS DETECTED" -ForegroundColor Red
        $tokenRiskySignIns | ForEach-Object {
            Write-Host "  Time:        $($_.CreatedDateTime)"
            Write-Host "  IP:          $($_.IpAddress)"
            Write-Host "  Resource:    $($_.ResourceDisplayName)"
            Write-Host "  Risk events: $($_.RiskEventTypes -join ', ')"
            Write-Host "  Risk level:  $($_.RiskLevelDuringSignIn)"
            Write-Host ""
        }
    } else {
        Write-Host "No token risk events detected." -ForegroundColor Green
    }

    # Look for same-session sign-ins from different IPs
    # This can indicate token replay from a different location
    Write-Host "=== IP Consistency Check ==="
    Write-Host "Checking for same-session access from multiple IPs...`n"

    $uniqueIPs = $signIns | Select-Object -ExpandProperty IpAddress -Unique
    if ($uniqueIPs.Count -gt 3) {
        Write-Host "⚠ HIGH IP DIVERSITY: $($uniqueIPs.Count) unique IPs in $HoursBack hours" -ForegroundColor Red
        Write-Host "This may indicate token theft and replay from multiple attacker systems."
        Write-Host "IPs seen: $($uniqueIPs -join ', ')"
    } else {
        Write-Host "IP diversity normal: $($uniqueIPs.Count) unique IPs" -ForegroundColor Green
        Write-Host "IPs: $($uniqueIPs -join ', ')"
    }

    # Check for CAE revocation events
    Write-Host "`n=== CAE Revocation Events ==="
    $caeEvents = $signIns | Where-Object {
        $_.AppliedConditionalAccessPolicies |
        Where-Object { $_.EnforcedGrantControls -contains "continuousAccessEvaluation" }
    }

    if ($caeEvents) {
        Write-Host "CAE events found: $($caeEvents.Count)"
        Write-Host "CAE revocation is working — token use was interrupted by real-time evaluation."
    } else {
        Write-Host "No CAE events in this window." -ForegroundColor Yellow
    }
}

Find-TokenAnomalies -UserPrincipalName "user@yourtenant.onmicrosoft.com" -HoursBack 24
```

---

### Step 2 — Emergency Token Revocation

```powershell
function Invoke-EmergencyTokenRevocation {
    param(
        [Parameter(Mandatory)]
        [string]$UserId,
        [Parameter(Mandatory)]
        [string]$UserPrincipalName,
        [string]$Reason = "Suspected token theft",
        [string]$IncidentTicket = "INC-PENDING"
    )

    Write-Host "`n=== EMERGENCY TOKEN REVOCATION ===" -ForegroundColor Red
    Write-Host "User:    $UserPrincipalName"
    Write-Host "Reason:  $Reason"
    Write-Host "Ticket:  $IncidentTicket"
    Write-Host "Time:    $(Get-Date)`n"

    # Step 1 — Revoke all sign-in sessions
    Write-Host "Revoking sign-in sessions..." -ForegroundColor Yellow
    $result = Invoke-MgGraphRequest -Method POST `
        -Uri "https://graph.microsoft.com/v1.0/users/$UserId/revokeSignInSessions"
    Write-Host "✓ Sign-in sessions revoked" -ForegroundColor Green
    Write-Host "  CAE-capable apps (Exchange, Teams, SharePoint): token revoked in seconds"
    Write-Host "  Non-CAE apps: token valid until expiry (up to 60-90 min)"

    # Step 2 — Confirm as compromised in Identity Protection
    Write-Host "`nElevating user risk to High..." -ForegroundColor Yellow
    Invoke-MgGraphRequest -Method POST `
        -Uri "https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/confirmCompromised" `
        -Body (@{ userIds = @($UserId) } | ConvertTo-Json) `
        -ContentType "application/json"
    Write-Host "✓ User risk set to High" -ForegroundColor Green
    Write-Host "  User must complete password reset before regaining access"

    # Step 3 — Force password change at next sign-in
    Write-Host "`nRequiring password change at next sign-in..." -ForegroundColor Yellow
    Update-MgUser -UserId $UserId `
        -PasswordProfile @{ ForceChangePasswordNextSignIn = $true }
    Write-Host "✓ Password change required" -ForegroundColor Green

    # Step 4 — Log the action
    Write-Host "`n=== REVOCATION COMPLETE ===" -ForegroundColor Green
    Write-Host "All three actions completed at $(Get-Date):"
    Write-Host "  1. Sessions revoked"
    Write-Host "  2. User risk elevated to High"
    Write-Host "  3. Password change required at next sign-in"
    Write-Host "`nNext steps:"
    Write-Host "  - Monitor for re-authentication attempts from suspicious IPs"
    Write-Host "  - Run Get-UserAuditTrail to check what account accessed during suspected theft window"
    Write-Host "  - Issue Temporary Access Pass for legitimate user to regain access safely"
    Write-Host "  - Consider upgrading MFA to FIDO2 post-recovery (token theft is often preceded by phishing)"
}
```

---

### What You Learned
- Token theft attack detection signals — anomalousToken risk event type, high IP diversity, CAE events
- The difference between CAE-capable apps (near-instant revocation) and non-CAE apps (up to 90 minute gap)
- The three-step emergency response: revoke sessions, confirm compromised, force password change

### Interview Talking Point After This Lab
*"Token theft is dangerous specifically because it bypasses MFA — the attacker never needs to authenticate, they just replay a stolen access token. The detection signal is the anomalousToken risk event type in Identity Protection, or high IP diversity in the sign-in logs — the same session appearing from multiple IPs. Containment is revokeSignInSessions via Graph — for CAE-capable apps like Exchange and Teams that is near-instant revocation. For non-CAE apps the access token remains valid for up to 90 minutes, which is why Continuous Access Evaluation matters so much in a financial environment. You cannot accept a 90-minute window where a known-stolen token is still valid."*

---

## Lab 8 — Full Incident Response Simulation

### What You Will Learn
- How to run a complete end-to-end IR workflow from alert to closure
- How to produce an IR report that satisfies a regulated environment audit requirement
- How to document the timeline, containment actions, and post-incident hardening

### Why This Matters for BoC
This lab ties every previous lab together into a single runbook. In a BoC interview when they ask "walk me through an incident response," this is the story you tell.

---

### The Simulated Scenario

```
Alert fires in Sentinel at 02:17am:
"High-risk sign-in detected — Global Reader role account"

The alert shows:
- User: ops-monitoring@yourtenant.onmicrosoft.com
- IP: unfamiliar, geolocation outside Canada
- Risk level: High
- Risk event types: impossibleTravel, anomalousToken
- PIM activation: none (account has no eligible PIM roles)
- Time since alert: 8 minutes
```

---

### Step 1 — Immediate Triage (First 5 Minutes)

```powershell
# Source your IR toolkit
. .\Connect-IRSession.ps1
. .\IR-Functions.ps1

# Immediate triage — read before you act
$targetUPN = "ops-monitoring@yourtenant.onmicrosoft.com"

Write-Host "=== INCIDENT RESPONSE INITIATED ===" -ForegroundColor Red
Write-Host "Time: $(Get-Date)" -ForegroundColor Red
Write-Host "Alert: High-risk sign-in — anomalousToken + impossibleTravel`n" -ForegroundColor Red

# Full triage in one command
Invoke-UserTriage -UserPrincipalName $targetUPN -SignInDaysBack 7 -AuditHoursBack 48
```

---

### Step 2 — Scope Assessment (Minutes 5-10)

```powershell
# Check if the suspicious IP hit other accounts
$suspiciousIP = "x.x.x.x"  # Replace with IP from the alert
$startDate = (Get-Date).AddHours(-4).ToString("yyyy-MM-ddTHH:mm:ssZ")

$otherAccountsFromIP = Get-MgAuditLogSignIn `
    -Filter "ipAddress eq '$suspiciousIP' and createdDateTime ge $startDate" `
    -All `
    -Property "userPrincipalName,status,createdDateTime" |
    Where-Object { $_.UserPrincipalName -ne $targetUPN }

if ($otherAccountsFromIP) {
    Write-Host "⚠ SCOPE WIDER THAN SINGLE ACCOUNT" -ForegroundColor Red
    Write-Host "Same IP hit these accounts:"
    $otherAccountsFromIP | Select-Object UserPrincipalName, CreatedDateTime |
        Format-Table -AutoSize
} else {
    Write-Host "Scope appears limited to single account." -ForegroundColor Yellow
}

# Check for App Registration credential changes in same window
Find-AppRegistrationCredentialChanges -HoursBack 4
```

---

### Step 3 — Containment Decision

```powershell
# Based on triage findings, decide containment level

# For this scenario — High risk, anomalousToken, impossibleTravel
# = Revoke sessions immediately, flag for investigation before disabling

$user = Get-MgUser -Filter "userPrincipalName eq '$targetUPN'" -Property "id"

Write-Host "`n=== CONTAINMENT DECISION ===" -ForegroundColor Red
Write-Host "Risk: HIGH | Signals: anomalousToken + impossibleTravel"
Write-Host "Privilege level: Global Reader (read-only — no write capability)"
Write-Host "Decision: Revoke sessions + elevate risk. Monitor before disabling."
Write-Host "Rationale: Read-only role limits blast radius. Disable only if further"
Write-Host "           evidence of active threat or write-capable role activation.`n"

# Revoke sessions
Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/users/$($user.Id)/revokeSignInSessions"
Write-Host "✓ Sessions revoked at $(Get-Date)" -ForegroundColor Green

# Confirm compromised to force remediation
Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/confirmCompromised" `
    -Body (@{ userIds = @($user.Id) } | ConvertTo-Json) `
    -ContentType "application/json"
Write-Host "✓ User risk elevated to High at $(Get-Date)" -ForegroundColor Green
```

---

### Step 4 — Generate the IR Report

```powershell
function New-IncidentReport {
    param(
        [Parameter(Mandatory)]
        [string]$IncidentId,
        [Parameter(Mandatory)]
        [string]$AffectedUser,
        [Parameter(Mandatory)]
        [string]$AlertType,
        [string]$AttackerIP,
        [string[]]$ContainmentActions,
        [string[]]$PostIncidentRecommendations
    )

    $reportTime = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $reportContent = @"
# IAM Incident Report
## Incident ID: $IncidentId
## Generated: $reportTime

---

## 1. Incident Summary
| Field | Value |
|---|---|
| Affected Identity | $AffectedUser |
| Alert Type | $AlertType |
| Attacker IP | $($AttackerIP ?? "Under investigation") |
| Detection Time | $reportTime |
| Reporter | IAM First Responder |

---

## 2. Triage Findings
$(Invoke-UserTriage -UserPrincipalName $AffectedUser 2>&1 | Out-String)

---

## 3. Containment Actions Taken
$(($ContainmentActions | ForEach-Object { "- $_ at $(Get-Date -Format 'HH:mm:ss')" }) -join "`n")

---

## 4. Scope Assessment
- Single account or multi-account: Under investigation
- App Registration credential changes: None detected in 4-hour window
- PIM activations during window: None detected

---

## 5. Post-Incident Recommendations
$(($PostIncidentRecommendations | ForEach-Object { "- $_" }) -join "`n")

---

## 6. Sign-Off
| Role | Name | Date |
|---|---|---|
| IAM First Responder | $($env:USERNAME) | $(Get-Date -Format 'yyyy-MM-dd') |
| Security Team Lead | [Pending] | |
| Manager | [Pending] | |
"@

    $filename = "IR-Report-$IncidentId-$(Get-Date -Format 'yyyyMMdd-HHmm').md"
    $reportContent | Out-File -FilePath $filename -Encoding UTF8
    Write-Host "`n✓ IR Report saved: $filename" -ForegroundColor Green
    return $filename
}

# Generate the report
New-IncidentReport `
    -IncidentId "INC-2026-001" `
    -AffectedUser "ops-monitoring@yourtenant.onmicrosoft.com" `
    -AlertType "anomalousToken + impossibleTravel — High risk sign-in" `
    -AttackerIP "x.x.x.x" `
    -ContainmentActions @(
        "Revoked all sign-in sessions via Graph API",
        "Confirmed user as compromised — risk elevated to High",
        "Password reset required at next sign-in"
    ) `
    -PostIncidentRecommendations @(
        "Enable number matching on Microsoft Authenticator for this account",
        "Consider upgrading to FIDO2 — anomalousToken indicates potential AiTM phishing",
        "Review Named Location policy — add this IP range to block list",
        "Investigate whether user received a phishing email in the 2 hours prior to alert",
        "Review CA policy coverage for Global Reader role accounts — device compliance not currently enforced"
    )
```

---

### What You Learned
- How to run a complete structured IR workflow from alert to documented closure
- How to produce an IR report that satisfies regulated environment audit requirements
- How to make and document the containment decision — with rationale based on privilege level and risk signals

### Interview Talking Point After This Lab
*"The IR report is as important as the IR action in a regulated environment. BoC's auditors will not just ask what we did — they will ask for the evidence that we followed a structured process. My IR runbook ends with a generated report that captures the timeline, triage findings, containment actions with timestamps, scope assessment, and post-incident recommendations. The rationale for why I chose to revoke sessions rather than disable the account is documented — privilege level was read-only so the blast radius was limited and a disable would have caused legitimate disruption. That documented decision-making is what distinguishes a mature IR process from a reactive one."*

---

## Lab Completion Summary

### What You Can Now Say You Have Done

| Capability | Evidence from Labs |
|---|---|
| Tenant-wide risky sign-in scanning | Lab 2 |
| Impossible travel detection with physics validation | Lab 2 |
| MFA fatigue pattern detection and containment | Lab 3 |
| Credential stuffing cross-account detection | Lab 4 |
| App Registration persistence hunting | Lab 5 |
| PIM activation anomaly detection | Lab 6 |
| Token theft detection and emergency revocation | Lab 7 |
| End-to-end IR simulation with report generation | Lab 8 |

### The Interview Answer After These Labs

*"As an IAM first responder I use a structured PowerShell toolkit built on Microsoft Graph. The first five minutes of any identity incident cover four questions: what is the user's current risk state from Identity Protection, what privilege level does the account have, what does the sign-in history show in terms of IP and location patterns, and what directory changes did the account make. Those four answers tell me urgency, blast radius, and scope before I take any containment action. From there I follow an escalating containment ladder — session revocation first, account disable only if the account has write-capable roles or there is evidence of active attacker access. Every action is timestamped and documented in an IR report because in a regulated environment the evidence trail is as important as the containment itself."*

---

*End of IR Labs Document*

---
**Kamran Arif | IAM Engineer | SC-300 | AZ-305 | AZ-104**
**kamranarif.ca@outlook.com | linkedin.com/in/karifa | 905-906-2786**
