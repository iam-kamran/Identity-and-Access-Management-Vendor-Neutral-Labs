# Hands-On IAM Labs — Closing the Gaps
### Kamran Arif — CI/CD Pipeline Identity + AI Agent Identity
### Microsoft Entra ID P2 | Azure DevOps Free Tier | Azure Free Subscription

---

**Author:** Kamran Arif  - https://www.linkedin.com/in/karifa/

> **Before You Start — Environment Setup**
>
> You need three things active before running any lab:
>
> 1. **Entra ID P2 Trial or License** — confirm at portal.azure.com → Entra ID → Licenses → All Products. You should see Microsoft Entra ID P2 listed.
> 2. **Azure DevOps Free Organisation** — sign up at dev.azure.com. Free tier gives you 1 parallel pipeline job and 5 free users. Enough for every lab here.
> 3. **Azure Subscription** — Free tier (portal.azure.com → Subscriptions → Add → Free Trial) gives you $200 credit. All labs here are designed to stay well within free tier limits.
>
> **Estimated cost across all 10 labs: under $5 if you clean up resources after each lab.**
>
> **Time per lab: 45–90 minutes.**

---

## Table of Contents

### Section A — CI/CD Pipeline Identity Labs
- [Lab 1 — Create an Azure DevOps Organisation and Your First Pipeline](#lab-1--create-an-azure-devops-organisation-and-your-first-pipeline)
- [Lab 2 — Service Connection with Workload Identity Federation (No Secrets)](#lab-2--service-connection-with-workload-identity-federation-no-secrets)
- [Lab 3 — Pull Secrets from Key Vault in a Pipeline Using Managed Identity](#lab-3--pull-secrets-from-key-vault-in-a-pipeline-using-managed-identity)
- [Lab 4 — Deploy an Entra ID App Registration from a Pipeline (IAM as Code)](#lab-4--deploy-an-entra-id-app-registration-from-a-pipeline-iam-as-code)
- [Lab 5 — Audit and Govern Pipeline Identities Using Entra ID and Access Reviews](#lab-5--audit-and-govern-pipeline-identities-using-entra-id-and-access-reviews)

### Section B — AI Agent Identity Labs
- [Lab 6 — Register Your First AI Agent Identity in Entra Agent ID](#lab-6--register-your-first-ai-agent-identity-in-entra-agent-id)
- [Lab 7 — Scope AI Agent Access with Least-Privilege OAuth2 Permissions](#lab-7--scope-ai-agent-access-with-least-privilege-oauth2-permissions)
- [Lab 8 — Apply Conditional Access for Workload Identities to an AI Agent](#lab-8--apply-conditional-access-for-workload-identities-to-an-ai-agent)
- [Lab 9 — Govern AI Agent Lifecycle with Access Reviews and Lifecycle Workflows](#lab-9--govern-ai-agent-lifecycle-with-access-reviews-and-lifecycle-workflows)
- [Lab 10 — Monitor and Audit AI Agent Sign-In Activity in Sentinel](#lab-10--monitor-and-audit-ai-agent-sign-in-activity-in-sentinel)

---

## SECTION A — CI/CD PIPELINE IDENTITY LABS

---

## Lab 1 — Create an Azure DevOps Organisation and Your First Pipeline

### What You Will Learn
- How Azure DevOps is structured (organisation → project → repo → pipeline)
- How a YAML pipeline works — stages, jobs, steps
- How a pipeline authenticates to Azure (the problem that Labs 2 and 3 solve)

### Why This Matters for BoC
You told the BoC panel that you understand the CI/CD identity architecture. This lab gives you the hands-on foundation to back that up. Understanding pipeline structure is what separates someone who can speak to the pattern from someone who has actually built one.

### Reference Learning
- 📖 Microsoft Learn: [Create your first pipeline](https://learn.microsoft.com/en-us/azure/devops/pipelines/create-first-pipeline)
- 🎥 YouTube: [Azure DevOps Pipeline Tutorial for Beginners — TechWorld with Nana](https://www.youtube.com/watch?v=4BibQ69MD8c)

---

### Step-by-Step Instructions

**Step 1 — Create an Azure DevOps Organisation**

1. Go to [dev.azure.com](https://dev.azure.com)
2. Sign in with the same Microsoft account linked to your Entra ID P2 tenant
3. Click **New organisation** → follow the prompts
4. Name it something memorable — `KamranIAMLabs` works
5. Choose a region close to you (Canada Central)

**Step 2 — Create a Project**

1. Inside your organisation click **New project**
2. Name: `IAM-Pipeline-Labs`
3. Visibility: **Private**
4. Version control: **Git**
5. Click **Create**

**Step 3 — Create a Repository with Sample Code**

1. In the left sidebar click **Repos** → **Files**
2. Click **Initialize** to create a default README
3. Click **New file** → name it `azure-pipelines.yml`
4. Paste this content:

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: IdentityCheck
    displayName: 'Lab 1 - Identity Check Stage'
    jobs:
      - job: WhoAmI
        displayName: 'Print Pipeline Identity'
        steps:
          - task: AzureCLI@2
            displayName: 'Check what identity the pipeline is using'
            inputs:
              azureSubscription: 'PLACEHOLDER - will fix in Lab 2'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                echo "=== Pipeline Identity Check ==="
                az account show
                az ad signed-in-user show 2>/dev/null || echo "Running as service principal"
                echo "=== End Identity Check ==="
```

5. Commit the file to main branch

**Step 4 — Understand the YAML Structure**

Before running anything, read through the file you just created. Every Azure DevOps pipeline YAML has this structure:

```
trigger          → which branch changes kick off the pipeline
pool             → what machine runs the pipeline (ubuntu-latest = free hosted agent)
stages           → top-level groupings (can have multiple stages: Build, Test, Deploy)
  jobs           → units of work within a stage (run in parallel or sequence)
    steps        → individual tasks within a job (sequential)
      task       → a specific action (AzureCLI@2, PowerShell@2, CopyFiles@2 etc)
```

**Step 5 — Explore the Pipeline Runs Interface**

1. In left sidebar click **Pipelines** → **Pipelines**
2. You will see the pipeline you just created but it will fail because `PLACEHOLDER` is not a real service connection
3. Click into the failed run → read the error message
4. Note: the error says it cannot find a service connection called `PLACEHOLDER`
5. This is exactly what Lab 2 fixes

---

### What You Learned

- Azure DevOps structure: organisation → project → repo → pipeline
- YAML pipeline anatomy: trigger, pool, stages, jobs, steps, tasks
- Why a pipeline needs a service connection to talk to Azure
- The pipeline failed because it has no identity to authenticate with — Lab 2 solves this

### Interview Talking Point After This Lab
*"I've set up Azure DevOps organisations and written YAML pipelines from scratch. The key concept I understand is that a pipeline is just another workload that needs an identity — and the right way to give it one is through a Workload Identity Federation service connection, not a stored secret."*

---

## Lab 2 — Service Connection with Workload Identity Federation (No Secrets)

### What You Will Learn
- How to create an Azure DevOps service connection using Workload Identity Federation
- What happens in Entra ID when you create this connection (App Registration + federated credential)
- How to verify the pipeline authenticates with no stored secret

### Why This Matters for BoC
This is the exact pattern BoC will use — and the exact gap from your previous interview. After this lab you can say "I have configured Workload Identity Federation service connections in Azure DevOps" with full accuracy.

### Reference Learning
- 📖 Microsoft Learn: [Connect to Azure with Workload Identity Federation](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure)
- 📖 Microsoft Learn: [Workload identity federation overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- 🎥 YouTube: [Workload Identity Federation in Azure DevOps — John Savill](https://www.youtube.com/watch?v=PjRbMJiJFV8)

---

### Step-by-Step Instructions

**Step 1 — Create the Service Connection**

1. In Azure DevOps go to your project `IAM-Pipeline-Labs`
2. Click **Project settings** (bottom left cog icon)
3. Under **Pipelines** click **Service connections**
4. Click **New service connection**
5. Select **Azure Resource Manager** → click **Next**
6. Select **App registration (automatic)** with credential type **Workload identity federation**
7. Click **Next**
8. Select your Azure **Subscription** from the dropdown
9. Leave **Resource group** blank (subscription scope for now)
10. Name it: `AzureWIF-ServiceConnection`
11. Check **Grant access permission to all pipelines**
12. Click **Save**

**Step 2 — Inspect What Was Created in Entra ID**

Azure DevOps just created an App Registration automatically. Go inspect it:

1. Go to [entra.microsoft.com](https://entra.microsoft.com)
2. Navigate to **Applications** → **App registrations** → **All applications**
3. Search for a new app — it will have a name like `IAM-Pipeline-Labs-AzureWIF-ServiceConnection-<guid>`
4. Click on it
5. Go to **Certificates & secrets** → **Federated credentials** tab
6. You will see a federated credential with:
   - **Issuer**: `https://login.microsoftonline.com/<your-tenant-id>/v2.0`
   - **Subject**: contains your Azure DevOps organisation and project name
   - **Audience**: `api://AzureADTokenExchange`

**This is the trust relationship.** Azure DevOps presents an OIDC token for this subject → Entra ID exchanges it for an access token → pipeline authenticates to Azure. No secret anywhere.

**Step 3 — Check the RBAC Assignment**

1. Go to portal.azure.com → your Subscription
2. Click **Access control (IAM)** → **Role assignments**
3. Find the App Registration that was just created
4. Note it has **Contributor** role at subscription scope

> **IAM Insight:** In a real environment you would scope this down to a resource group, not the whole subscription. Contributor at subscription scope is too broad for a pipeline that only needs to deploy to one resource group. Remember this — it will come up in a BoC technical panel.

**Step 4 — Update Your Pipeline to Use the Service Connection**

1. Go back to your repo in Azure DevOps → Repos → Files
2. Edit `azure-pipelines.yml`
3. Replace `PLACEHOLDER - will fix in Lab 2` with `AzureWIF-ServiceConnection`
4. Full updated file:

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: IdentityCheck
    displayName: 'Lab 2 - Workload Identity Federation'
    jobs:
      - job: WhoAmI
        displayName: 'Verify pipeline identity - no secrets'
        steps:
          - task: AzureCLI@2
            displayName: 'Show authenticated identity'
            inputs:
              azureSubscription: 'AzureWIF-ServiceConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                echo "=== Authenticated via Workload Identity Federation ==="
                echo "Account details:"
                az account show --output table
                echo ""
                echo "Service Principal details:"
                az ad sp show --id $(az account show --query user.name -o tsv) \
                  --query "{DisplayName:displayName, AppId:appId}" \
                  --output table 2>/dev/null || echo "Identity confirmed via WIF token exchange"
                echo "=== No client secret was used in this authentication ==="
```

5. Commit the change
6. Go to **Pipelines** → watch it run
7. Click into the job → expand the **Show authenticated identity** step
8. You will see the subscription details and confirmation the pipeline ran without a secret

---

### What You Learned

- How to create an Azure DevOps Workload Identity Federation service connection
- What Entra ID creates in the background — App Registration + federated credential
- How the trust works — OIDC token from Azure DevOps exchanged for Entra ID access token
- The RBAC assignment that the service connection uses to act on Azure resources

### Interview Talking Point After This Lab
*"I've configured Workload Identity Federation service connections in Azure DevOps. The connection creates an App Registration in Entra ID with a federated credential — the pipeline presents an OIDC token, Entra ID validates the subject against the federated credential and exchanges it for an access token. No client secret, no rotation, no expiry risk. In a production environment I would scope the RBAC assignment to the specific resource group the pipeline deploys to, not the entire subscription."*

---

## Lab 3 — Pull Secrets from Key Vault in a Pipeline Using Managed Identity

### What You Will Learn
- How to create an Azure Key Vault and store a secret
- How to give a pipeline identity access to Key Vault using RBAC
- How to pull a secret from Key Vault in a pipeline without storing it as a pipeline variable

### Why This Matters for BoC
Secret management in pipelines is a real BoC requirement. This lab gives you the hands-on story: *"Pipelines retrieve secrets from Key Vault at runtime — nothing stored in Azure DevOps variable groups as plaintext."*

### Reference Learning
- 📖 Microsoft Learn: [Use Azure Key Vault secrets in pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/azure-key-vault)
- 🎥 YouTube: [Azure Key Vault in Azure DevOps Pipelines — Microsoft Developer](https://www.youtube.com/watch?v=S3MoF5KHNWQ)

---

### Step-by-Step Instructions

**Step 1 — Create a Key Vault**

1. Go to portal.azure.com
2. Search for **Key vaults** → **Create**
3. Resource group: create new → `rg-iam-labs`
4. Key vault name: `kv-iamlabs-<your-initials>` (must be globally unique)
5. Region: Canada Central
6. Pricing tier: **Standard**
7. **Permission model: Azure role-based access control** ← important, select this
8. Click **Review + create** → **Create**

**Step 2 — Add a Secret**

1. Once created go to your Key Vault
2. Click **Secrets** → **Generate/Import**
3. Name: `pipeline-demo-secret`
4. Secret value: `SuperSecretDatabasePassword123` (fake value for the lab)
5. Click **Create**

**Step 3 — Grant the Pipeline Identity Access to Key Vault**

Your pipeline authenticates as the App Registration created in Lab 2. Give it read access to secrets:

1. In your Key Vault click **Access control (IAM)**
2. Click **Add role assignment**
3. Role: **Key Vault Secrets User**
4. Assign access to: **User, group, or service principal**
5. Search for the App Registration name from Lab 2 (`IAM-Pipeline-Labs-AzureWIF-ServiceConnection-<guid>`)
6. Select it → **Review + assign**

**Step 4 — Create a Variable Group Linked to Key Vault**

1. In Azure DevOps go to **Pipelines** → **Library**
2. Click **+ Variable group**
3. Name: `KeyVault-Secrets`
4. Toggle **Link secrets from an Azure key vault as variables** → ON
5. Azure subscription: `AzureWIF-ServiceConnection`
6. Key vault name: select `kv-iamlabs-<your-initials>`
7. Click **+ Add** under Variables
8. Select `pipeline-demo-secret` from the list
9. Click **OK** → **Save**

**Step 5 — Update Pipeline to Use Key Vault Secret**

Edit `azure-pipelines.yml`:

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: KeyVault-Secrets

stages:
  - stage: SecretDemo
    displayName: 'Lab 3 - Key Vault Secret Retrieval'
    jobs:
      - job: UseSecret
        displayName: 'Pull secret from Key Vault - no plaintext in pipeline'
        steps:
          - task: AzureCLI@2
            displayName: 'Verify Key Vault access via pipeline identity'
            inputs:
              azureSubscription: 'AzureWIF-ServiceConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                echo "=== Key Vault Secret Retrieval ==="
                echo "Secret was retrieved from Key Vault by the pipeline identity"
                echo "Secret length: ${#PIPELINE_DEMO_SECRET} characters"
                echo "First 3 chars: ${PIPELINE_DEMO_SECRET:0:3}***"
                echo "=== Secret never stored in Azure DevOps variables ==="
            env:
              PIPELINE_DEMO_SECRET: $(pipeline-demo-secret)
```

Commit and run. The pipeline retrieves the secret from Key Vault at runtime — it is never stored in Azure DevOps.

**Step 6 — Verify the Audit Trail in Key Vault**

1. Go to your Key Vault in Azure portal
2. Click **Insights** or **Monitoring** → **Diagnostic settings**
3. Note: in a production environment you would send Key Vault audit logs to Log Analytics
4. Every secret retrieval by the pipeline would appear in these logs — who retrieved what and when

---

### What You Learned

- How to create a Key Vault and store secrets using RBAC permission model
- How to assign Key Vault Secrets User role to a pipeline identity (least privilege)
- How to link an Azure DevOps variable group to Key Vault
- How secrets flow at runtime — Key Vault → pipeline environment variable — without ever being stored in Azure DevOps

### Interview Talking Point After This Lab
*"Pipeline secrets should never be stored as Azure DevOps pipeline variables in plaintext. I've configured variable groups linked to Azure Key Vault — the pipeline identity has Key Vault Secrets User role scoped to that vault, and secrets are retrieved at runtime. The audit trail in Key Vault shows every retrieval with a timestamp and the identity that accessed it."*

---

## Lab 4 — Deploy an Entra ID App Registration from a Pipeline (IAM as Code)

### What You Will Learn
- How to use a pipeline to create and configure an Entra ID App Registration
- How to use Microsoft Graph API calls from a pipeline
- The concept of IAM configuration as code — repeatable, auditable, version-controlled

### Why This Matters for BoC
IAM as code is the mature answer to "how do you ensure consistent App Registration configuration across environments." This lab gives you a concrete example you can describe — deploying an App Registration with specific API permissions via a pipeline.

### Reference Learning
- 📖 Microsoft Learn: [Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)
- 📖 Microsoft Learn: [Create App Registration with Graph API](https://learn.microsoft.com/en-us/graph/api/application-post-applications)
- 🎥 YouTube: [Microsoft Graph API Tutorial — Corey Schafer style walkthrough by Andy Malone MVP](https://www.youtube.com/watch?v=Lk1gJTmVGO4)

---

### Step-by-Step Instructions

**Step 1 — Grant the Pipeline Identity Graph API Permissions**

The pipeline's App Registration needs permission to create other App Registrations via Graph API:

1. Go to Entra admin centre → **App registrations** → find the pipeline App Registration
2. Click **API permissions** → **Add a permission**
3. Select **Microsoft Graph** → **Application permissions**
4. Add: `Application.ReadWrite.OwnedBy`
   - This allows the pipeline to create and manage App Registrations it owns — least privilege, not `Application.ReadWrite.All`
5. Click **Add permissions**
6. Click **Grant admin consent for <your tenant>** → confirm

**Step 2 — Create the Pipeline YAML for App Registration Deployment**

Create a new file in your repo called `deploy-app-registration.yml`:

```yaml
trigger: none  # Manual trigger only for this lab

pool:
  vmImage: 'ubuntu-latest'

parameters:
  - name: appName
    displayName: 'App Registration Name'
    type: string
    default: 'lab4-demo-app'
  - name: environment
    displayName: 'Environment'
    type: string
    default: 'dev'
    values:
      - dev
      - test
      - prod

stages:
  - stage: DeployAppRegistration
    displayName: 'Lab 4 - IAM as Code - Deploy App Registration'
    jobs:
      - job: CreateApp
        displayName: 'Create App Registration via Graph API'
        steps:
          - task: AzurePowerShell@5
            displayName: 'Deploy App Registration'
            inputs:
              azureSubscription: 'AzureWIF-ServiceConnection'
              ScriptType: 'InlineScript'
              Inline: |
                # Install Graph module
                Install-Module Microsoft.Graph -Scope CurrentUser -Force -AllowClobber

                # Get access token for Graph API using the pipeline's identity
                $token = (Get-AzAccessToken -ResourceUrl "https://graph.microsoft.com").Token
                Connect-MgGraph -AccessToken ($token | ConvertTo-SecureString -AsPlainText -Force)

                $appName = "${{ parameters.appName }}-${{ parameters.environment }}"

                Write-Host "=== Checking if App Registration already exists ==="
                $existingApp = Get-MgApplication -Filter "displayName eq '$appName'" -ErrorAction SilentlyContinue

                if ($existingApp) {
                    Write-Host "App Registration '$appName' already exists. AppId: $($existingApp.AppId)"
                    Write-Host "Skipping creation - idempotent deployment"
                } else {
                    Write-Host "=== Creating App Registration: $appName ==="
                    $app = New-MgApplication -DisplayName $appName -SignInAudience "AzureADMyOrg" -Notes "Created by Azure DevOps pipeline on $(Get-Date)"

                    Write-Host "App Registration created successfully"
                    Write-Host "Display Name: $($app.DisplayName)"
                    Write-Host "Application (client) ID: $($app.AppId)"
                    Write-Host "Object ID: $($app.Id)"

                    # Set the app ID URI
                    Update-MgApplication -ApplicationId $app.Id -IdentifierUris @("api://$($app.AppId)")
                    Write-Host "App ID URI set to: api://$($app.AppId)"
                }

                Write-Host "=== Deployment Complete ==="
                Write-Host "Environment: ${{ parameters.environment }}"
                Write-Host "Pipeline run initiated by: $(Build.RequestedFor)"
              azurePowerShellVersion: 'LatestVersion'
```

**Step 3 — Run the Pipeline Manually**

1. Go to **Pipelines** → find `deploy-app-registration.yml`
2. Click **Run pipeline**
3. Fill in parameters:
   - App Registration Name: `lab4-demo-app`
   - Environment: `dev`
4. Click **Run**
5. Watch the pipeline create the App Registration

**Step 4 — Verify in Entra ID**

1. Go to Entra admin centre → **App registrations** → **All applications**
2. Search for `lab4-demo-app-dev`
3. Confirm it was created with the correct settings
4. Check the **Notes** field — it should say "Created by Azure DevOps pipeline on <timestamp>"

**Step 5 — Run Again — Observe Idempotency**

1. Run the pipeline again with the same parameters
2. Observe that it detects the existing App Registration and skips creation
3. No duplicate is created — the pipeline is idempotent

> **Key IAM concept:** Idempotent deployments are critical in IAM automation. Running the same pipeline twice should not create duplicate App Registrations or change something that should not change. Always check for existing resources before creating.

**Step 6 — Clean Up**

1. Go to Entra admin centre → App registrations → find `lab4-demo-app-dev`
2. Delete it to keep your tenant clean

---

### What You Learned

- How to call Microsoft Graph API from a pipeline using the pipeline's Workload Identity
- How to deploy App Registrations as code — repeatable and version-controlled
- The concept of idempotent IAM automation — check before create
- How pipeline-initiated changes appear in Entra audit logs with the pipeline identity as the actor

### Interview Talking Point After This Lab
*"I've implemented IAM configuration as code — using Azure DevOps pipelines to deploy and manage App Registrations via Microsoft Graph API. The pipeline authenticates via Workload Identity Federation, calls Graph with application permissions scoped to apps it owns, and the deployment is idempotent — it checks for existing resources before creating. Every change is in source control and the Entra audit log shows the pipeline identity as the actor."*

---

## Lab 5 — Audit and Govern Pipeline Identities Using Entra ID and Access Reviews

### What You Will Learn
- How to find all service principals created by Azure DevOps in your tenant
- How to run an Access Review on service principals (a P2-only feature)
- How to apply Conditional Access for Workload Identities

### Why This Matters for BoC
Every pipeline you create in Azure DevOps creates an App Registration in Entra ID. Without governance, these accumulate. This lab shows you how to inventory and govern pipeline identities the same way you govern human identities.

### Reference Learning
- 📖 Microsoft Learn: [Access reviews for service principals](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-create-roles-and-resource-roles-review)
- 📖 Microsoft Learn: [Conditional Access for workload identities](https://learn.microsoft.com/en-us/entra/identity/conditional-access/workload-identity)
- 🎥 YouTube: [Access Reviews in Microsoft Entra ID — Peter Rising MVP](https://www.youtube.com/watch?v=CvpHoWRBiD4)

---

### Step-by-Step Instructions

**Step 1 — Inventory All Service Principals from Azure DevOps**

Run this in the Azure Cloud Shell (portal.azure.com → Cloud Shell icon):

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "Application.Read.All","Directory.Read.All"

# Find all service principals that look like Azure DevOps pipeline identities
$servicePrincipals = Get-MgServicePrincipal -All | Where-Object {
    $_.DisplayName -like "*IAM-Pipeline-Labs*" -or
    $_.DisplayName -like "*AzureWIF*" -or
    $_.Notes -like "*Azure DevOps*"
}

Write-Host "=== Pipeline Service Principals Found ==="
foreach ($sp in $servicePrincipals) {
    Write-Host "Name: $($sp.DisplayName)"
    Write-Host "App ID: $($sp.AppId)"
    Write-Host "Created: $($sp.AdditionalProperties.createdDateTime)"
    Write-Host "---"
}

# Also check App Registrations for any with AzureDevOps in notes
$apps = Get-MgApplication -All | Where-Object {
    $_.Notes -like "*pipeline*" -or $_.DisplayName -like "*IAM-Pipeline-Labs*"
}

Write-Host "=== App Registrations (pipeline-created) ==="
$apps | Select-Object DisplayName, AppId, CreatedDateTime | Format-Table
```

**Step 2 — Create an Access Review for Service Principals**

1. Go to [entra.microsoft.com](https://entra.microsoft.com)
2. Navigate to **Identity governance** → **Access reviews** → **New access review**
3. Select what to review: **Teams + Groups** — change to **Applications**
4. Scope: **Service principals**
5. Select your pipeline App Registration from the list
6. Review name: `Pipeline-Identity-Quarterly-Review`
7. Reviewers: **Specific users** → add yourself
8. Duration: 7 days
9. Recurrence: **Quarterly**
10. Upon completion settings:
    - Auto apply results: **Enable**
    - If reviewers do not respond: **No change** (for safety in lab)
11. Click **Start**

**Step 3 — Complete the Review**

1. You will receive an email notification about the pending review
2. Go to **myaccess.microsoft.com** to see the review queue
3. Alternatively in Entra admin centre → **Identity governance** → **Access reviews** → **My Reviews**
4. Find the review and click it
5. Review the pipeline service principal — click **Approve** (it is legitimate)
6. Add a justification: "Active pipeline identity for IAM Labs project — confirmed legitimate"
7. Click **Submit**

**Step 4 — Apply Conditional Access for Workload Identities (P2 Feature)**

1. Go to Entra admin centre → **Protection** → **Conditional Access**
2. Click **New policy**
3. Name: `Block-Pipeline-Identity-High-Risk`
4. Under **Users** → switch to **Workload identities**
5. Select **Service principals** → add your pipeline App Registration
6. Under **Conditions** → **Service principal risk**
   - Toggle to **Yes**
   - Select **High**
7. Under **Grant** → **Block access**
8. Enable policy: **Report-only** (do not enable for real — just observe)
9. Click **Save**

> **What this does:** If your pipeline service principal ever has high risk detected by Entra ID Protection, this CA policy would block it automatically. Same Zero Trust principle applied to non-human identities as to humans.

**Step 5 — View the Audit Trail**

1. Go to Entra admin centre → **Monitoring** → **Audit logs**
2. Filter by:
   - Date range: Last 24 hours
   - Service: **Access reviews**
3. You will see your Access Review decision logged with timestamp and reviewer
4. Also filter by Service: **Application management** to see the App Registration created in Lab 4

---

### What You Learned

- How to inventory pipeline service principals in your Entra ID tenant using Graph/PowerShell
- How to run Access Reviews on service principals (P2 feature — same governance applied to pipelines as to human access)
- How Conditional Access for Workload Identities extends Zero Trust to pipeline identities
- The audit trail that proves pipeline identity governance is in place

### Interview Talking Point After This Lab
*"I apply the same governance to pipeline identities as to human identities. Every service principal created by Azure DevOps is inventoried, reviewed quarterly through Entra ID Access Reviews for service principals, and has a Conditional Access policy that blocks it if service principal risk is detected as high. The governance model does not distinguish between human and non-human identities — both are governed at the identity layer."*

---

## SECTION B — AI AGENT IDENTITY LABS

---

## Lab 6 — Register Your First AI Agent Identity in Entra Agent ID

### What You Will Learn
- What Entra Agent ID is and how it differs from a standard App Registration
- How to register an agent identity in your tenant
- The concept of agent blueprints — templates for consistent agent security policies

### Why This Matters for BoC
Entra Agent ID is in public preview and BoC specifically mentioned AI agent identity as a current priority. Being able to say "I have worked with Entra Agent ID in a lab environment" puts you ahead of every candidate who only knows it exists.

### Reference Learning
- 📖 Microsoft Learn: [What is Microsoft Entra Agent ID](https://learn.microsoft.com/en-us/entra/agent-id/what-is-microsoft-entra-agent-id)
- 📖 Microsoft Learn: [Manage agent identities](https://learn.microsoft.com/en-us/entra/agent-id/how-to-view-manage-agent-identities)
- 🎥 YouTube: [Microsoft Entra Agent ID Overview — Microsoft Mechanics](https://www.youtube.com/results?search_query=Microsoft+Entra+Agent+ID+overview+2025)

---

### Step-by-Step Instructions

**Step 1 — Enable Entra Agent ID Preview**

Entra Agent ID is in public preview. Access it through the Entra admin centre:

1. Go to [entra.microsoft.com](https://entra.microsoft.com)
2. In the left sidebar look for **Agent ID** under the **Applications** section
3. If you do not see it, navigate to **Show more** → search for **Agent ID**
4. Alternatively go directly to: `https://entra.microsoft.com/#view/Microsoft_AAD_Agents/AgentsBlade`

> **Note:** If Agent ID is not yet visible in your tenant, you can simulate this lab using the App Registration approach below — register a service principal specifically tagged as an AI agent. The underlying identity constructs are the same.

**Step 2 — Create an Agent Blueprint**

An agent blueprint is a template — it defines the security policy applied to all instances of a particular agent type.

1. In Entra Agent ID click **Blueprints** → **New blueprint**
2. Name: `CustomerSupportAgent-Blueprint`
3. Description: `Blueprint for customer-facing AI support agents — read-only access to knowledge base`
4. Sponsor (human owner responsible for this agent type): assign yourself
5. Click **Create**

> **Key concept:** Every agent instance created from this blueprint inherits the same identity settings. If you have 1,000 instances of a customer support agent, the blueprint governs all 1,000 consistently. This is the scalability answer to AI agent identity.

**Step 3 — Register an Agent Identity**

1. Click **Agent identities** → **New agent identity**
2. Name: `CustomerSupportAgent-Instance-001`
3. Blueprint: select `CustomerSupportAgent-Blueprint`
4. Sponsor: yourself
5. Description: `Lab 6 demo agent — customer support bot`
6. Click **Register**

**Step 4 — Inspect the Agent Identity in App Registrations**

The agent identity is an App Registration under the hood — with additional metadata:

1. Go to **App registrations** → **All applications**
2. Find `CustomerSupportAgent-Instance-001`
3. Note the additional properties that mark it as an agent identity vs a standard App Registration
4. Click **Owners** — you are listed as the sponsor/owner
5. Click **Certificates & secrets** — no secrets added yet (good — you will add permissions in Lab 7)

**Step 5 — Compare Agent Identity vs Standard App Registration**

Open both side by side — your Lab 4 demo app and the new agent identity:

| Property | Standard App Reg | Agent Identity |
|---|---|---|
| Created via | App registrations UI | Agent ID blade |
| Blueprint | None | CustomerSupportAgent-Blueprint |
| Sponsor | Not required | Required — human accountability |
| Governance | Access Reviews optional | Lifecycle governed by blueprint |
| Intended workload | Any | AI agent specifically |

**Step 6 — View Agent Identity in Audit Logs**

1. Entra admin centre → **Monitoring** → **Audit logs**
2. Filter by: Service = **Application management**
3. You will see the agent identity registration event with your account as the initiator

---

### What You Learned

- What Entra Agent ID is — purpose-built identity constructs for AI agents, not just reused App Registrations
- Agent blueprints — templates that enforce consistent security policy across all instances of an agent type
- The required human sponsor — every agent identity needs an accountable human owner
- How agent identities appear in Entra ID audit logs

### Interview Talking Point After This Lab
*"I've worked with Entra Agent ID in a lab environment. The key difference from a standard App Registration is the required human sponsor and the blueprint concept — a blueprint is a security policy template applied consistently across all instances of a given agent type. If you have a thousand customer support bots, the blueprint governs all of them from a single policy definition. That scalability is what makes AI agent governance tractable."*

---

## Lab 7 — Scope AI Agent Access with Least-Privilege OAuth2 Permissions

### What You Will Learn
- How to assign specific Microsoft Graph permissions to an agent identity
- How the On-Behalf-Of (OBO) flow works — agent acting with user's permissions
- Why least-privilege matters more for AI agents than for static services

### Why This Matters for BoC
An AI agent that can do anything is the most dangerous identity in your environment. Scoping what it can access — and being able to explain why you chose specific permissions — is core IAM engineer work.

### Reference Learning
- 📖 Microsoft Learn: [OAuth2 On-Behalf-Of flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- 📖 Microsoft Learn: [Microsoft Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference)
- 🎥 YouTube: [OAuth 2.0 and OpenID Connect explained — OktaDev](https://www.youtube.com/watch?v=996OiexHze0)

---

### Step-by-Step Instructions

**Step 1 — Define What the Agent Needs to Do**

Before assigning permissions, define the agent's job:

> The `CustomerSupportAgent` needs to:
> - Read the user's name and email (to personalise responses)
> - Read knowledge base documents from SharePoint (read-only)
> - Create support tickets in a backend system via a custom API
>
> The `CustomerSupportAgent` must NOT:
> - Read emails
> - Access calendars
> - Write to any Microsoft 365 service
> - Access other users' data

This definition drives the permission selection. Writing this down before touching the portal is the IAM engineer discipline — permissions should follow a defined need, not be added speculatively.

**Step 2 — Add Application Permissions (Agent Acting as Itself)**

For scenarios where the agent acts without a user present:

1. Go to App registrations → `CustomerSupportAgent-Instance-001`
2. Click **API permissions** → **Add a permission**
3. Select **Microsoft Graph** → **Application permissions**
4. Add only:
   - `User.Read.All` — read user profiles (to look up the customer)
   - `Sites.Read.All` — read SharePoint sites (knowledge base, read-only)
5. Click **Add permissions**
6. Click **Grant admin consent** → confirm

**Step 3 — Configure Delegated Permissions (Agent Acting On-Behalf-Of User)**

For scenarios where the agent acts on behalf of a specific logged-in user:

1. Click **Add a permission** → **Microsoft Graph** → **Delegated permissions**
2. Add:
   - `User.Read` — read the signed-in user's profile only (not all users)
3. Click **Add permissions**

> **IAM Insight:** Application permissions apply when the agent acts as itself (background processing). Delegated permissions apply when the agent acts on behalf of a specific user (interactive scenarios). In both cases, grant the minimum needed. `User.Read` (one user) vs `User.Read.All` (all users) is a significant privilege difference.

**Step 4 — Test the Permission Scope in Graph Explorer**

1. Go to [developer.microsoft.com/en-us/graph/graph-explorer](https://developer.microsoft.com/en-us/graph/graph-explorer)
2. Sign in with your account
3. Run: `GET https://graph.microsoft.com/v1.0/me` — works (User.Read)
4. Run: `GET https://graph.microsoft.com/v1.0/users` — works (User.Read.All if consented)
5. Run: `GET https://graph.microsoft.com/v1.0/me/messages` — FAILS (Mail.Read not granted)

This failure is correct. The agent cannot read emails because you did not grant that permission. This is least-privilege working as designed.

**Step 5 — Document the Permission Decision**

In a regulated environment like BoC, every permission grant needs documentation. Create a simple record:

```markdown
## Agent Permission Register — CustomerSupportAgent

| Permission | Type | Justification | Granted Date | Reviewer |
|---|---|---|---|---|
| User.Read.All | Application | Read customer profile for personalisation | [today] | [your name] |
| Sites.Read.All | Application | Read knowledge base documents | [today] | [your name] |
| User.Read | Delegated | Read signed-in user profile in OBO flows | [today] | [your name] |

## Explicitly Rejected Permissions
| Permission | Reason Rejected |
|---|---|
| Mail.Read | Not required — agent does not read emails |
| Mail.Send | Not required — agent does not send emails |
| Files.ReadWrite.All | Read-only required — write access not justified |
```

Save this in your Azure DevOps repo — permission decisions should be version-controlled.

---

### What You Learned

- How to scope OAuth2 permissions for an AI agent — application vs delegated permissions
- The difference between an agent acting as itself vs acting on behalf of a user (OBO flow)
- How to use Graph Explorer to verify what an identity can and cannot access
- How to document permission decisions — the audit trail that regulators expect

### Interview Talking Point After This Lab
*"When assigning permissions to an AI agent identity, I start by defining what the agent actually needs to do before opening the portal. Application permissions for background processing, delegated permissions for on-behalf-of flows. I use Graph Explorer to verify the scope — confirming what the identity can access and confirming that permissions it should not have are actually rejected. Every permission grant is documented with a business justification."*

---

## Lab 8 — Apply Conditional Access for Workload Identities to an AI Agent

### What You Will Learn
- How to create a Conditional Access policy scoped specifically to a service principal (AI agent identity)
- How to enforce location-based and risk-based controls on non-human identities
- Why this is the senior-level answer to AI agent security that most candidates cannot give

### Why This Matters for BoC
This is the concept that you surfaced during interview prep that "most candidates will not give." This lab lets you say you have actually configured it, not just described it conceptually.

### Reference Learning
- 📖 Microsoft Learn: [Conditional Access for workload identities](https://learn.microsoft.com/en-us/entra/identity/conditional-access/workload-identity)
- 📖 Microsoft Learn: [Service principal risk in Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/workload-identity#create-a-conditional-access-policy)

> **Licence requirement:** Conditional Access for workload identities requires Entra ID P2. You have this — confirmed in your environment setup.

---

### Step-by-Step Instructions

**Step 1 — Create a Named Location for Trusted IP Ranges**

First create a Named Location representing where the agent should authenticate from (Azure data centre IP ranges — not user locations):

1. Entra admin centre → **Protection** → **Conditional Access** → **Named locations**
2. Click **New location** → **IP ranges location**
3. Name: `Azure-Datacenter-TrustedRange`
4. Add IP range: `20.38.0.0/16` (this is an Azure Canada Central IP range — representative)
5. Mark as **trusted location**: Yes
6. Click **Create**

> **Note:** In a real environment you would add the specific Azure region IP ranges where your AI agents run. Microsoft publishes these at [microsoft.com/en-us/download/details.aspx?id=56519](https://www.microsoft.com/en-us/download/details.aspx?id=56519)

**Step 2 — Create the Conditional Access Policy for the Agent Identity**

1. Entra admin centre → **Protection** → **Conditional Access**
2. Click **New policy**
3. Name: `AI-Agent-Location-Restriction`

**Users:**
- Click **Users** → switch to **Workload identities** tab
- Select **Service principals**
- Search for and add: `CustomerSupportAgent-Instance-001`

**Conditions:**
- Click **Locations**
  - Configure: **Yes**
  - Include: **Any location**
  - Exclude: `Azure-Datacenter-TrustedRange`

**Grant:**
- Select **Block access**

**Enable policy:**
- Set to **Report-only** ← important, do not enable for real in a lab

4. Click **Save**

> **What this policy does:** If the CustomerSupportAgent authenticates from any IP address NOT in the Azure datacentre IP range — for example if its credentials were stolen and used from a home network — this policy blocks the authentication. The agent should only ever run from Azure infrastructure.

**Step 3 — Create a Risk-Based Policy for the Agent**

1. Create another new Conditional Access policy
2. Name: `AI-Agent-Risk-Block`

**Users:** Same agent service principal

**Conditions:**
- Click **Service principal risk**
  - Configure: **Yes**
  - Select: **High**

**Grant:** Block access

**Enable policy:** Report-only

3. Save

> **What this policy does:** If Entra ID Protection detects high-risk behaviour for this service principal — unusual authentication patterns, token anomalies, suspicious API call patterns — the agent is blocked automatically without human intervention.

**Step 4 — Review the Combined Policy Set**

You now have two CA policies for this agent:
- Location: block if not from Azure datacentre IPs
- Risk: block if service principal risk is high

This means even if an attacker steals the agent's credentials, they face two barriers:
1. They are not calling from an Azure IP range
2. The anomalous authentication raises the service principal risk score

**Step 5 — Simulate What Report-Only Shows**

1. Entra admin centre → **Monitoring** → **Sign-in logs**
2. Filter by: Application = `CustomerSupportAgent-Instance-001`
3. If you have any sign-ins from Lab 7, click into one
4. Scroll to **Conditional Access** tab
5. You will see the report-only policies listed with what the outcome would have been

---

### What You Learned

- How to create Conditional Access policies scoped to specific service principals — not just human users
- Location-based restriction for AI agents — should only authenticate from Azure infrastructure
- Service principal risk-based blocking — automatic response to anomalous agent behaviour
- How report-only mode lets you validate policies before enforcement

### Interview Talking Point After This Lab
*"I've configured Conditional Access for workload identities specifically scoped to an AI agent service principal. Two policies: one that blocks authentication from any IP outside the Azure datacentre ranges — the agent should only ever run from Azure infrastructure — and one that auto-blocks if service principal risk is detected as high. This is the same Zero Trust principle applied to AI agents as to human users: verify explicitly, assume breach, enforce at the identity layer."*

---

## Lab 9 — Govern AI Agent Lifecycle with Access Reviews and Lifecycle Workflows

### What You Will Learn
- How to run an Access Review specifically on an AI agent's role assignments
- How to set up a Lifecycle Workflow to deactivate an agent identity
- The concept of agent sponsorship — every agent needs a human accountable owner

### Why This Matters for BoC
AI agent governance is not just about creation — it is about the full lifecycle. Agents that are no longer needed should be deactivated automatically, not left running with standing access. This lab shows the governance controls.

### Reference Learning
- 📖 Microsoft Learn: [Access reviews for applications and service principals](https://learn.microsoft.com/en-us/entra/id-governance/access-reviews-application-certification)
- 📖 Microsoft Learn: [Lifecycle Workflows overview](https://learn.microsoft.com/en-us/entra/id-governance/what-are-lifecycle-workflows)
- 🎥 YouTube: [Entra ID Governance — John Savill Technical Training](https://www.youtube.com/watch?v=qUGKr7SGQSA)

---

### Step-by-Step Instructions

**Step 1 — Assign a Role to the Agent (Something to Review)**

First give the agent a role so there is something to certify in the Access Review:

1. Go to your Azure subscription in portal.azure.com
2. Click **Access control (IAM)** → **Add role assignment**
3. Role: **Reader**
4. Assign to: Service principal → search for `CustomerSupportAgent-Instance-001`
5. Save

**Step 2 — Create an Access Review for the Agent's Role Assignment**

1. Entra admin centre → **Identity governance** → **Access reviews**
2. Click **New access review**
3. Review type: **Teams + Groups** — change to **Azure resource roles**
4. Scope: **Subscription** → select your subscription
5. Role: **Reader**
6. Scope down to: include only service principals
7. Review name: `AI-Agent-Role-Certification-Q1`
8. Reviewers: **Specific users** → add yourself as the agent sponsor/reviewer
9. Duration: 7 days
10. Recurrence: **Quarterly**
11. Auto-apply: **Enable**
12. If reviewers do not respond: **Deny access** ← important — agents should lose access if their sponsor does not re-certify
13. Click **Start**

> **Key governance principle:** For AI agents, the default for non-response should be deny/remove — not keep. An unreviewed agent is an ungoverned agent. In a regulated environment like BoC, an agent that cannot be certified by a human sponsor should not have access.

**Step 3 — Complete the Review as the Agent Sponsor**

1. Go to myaccess.microsoft.com
2. Find the review `AI-Agent-Role-Certification-Q1`
3. Review the agent — click **Approve**
4. Justification: "CustomerSupportAgent active and in production — Reader role required for knowledge base access. Reviewed by sponsor."
5. Submit

**Step 4 — Create a Lifecycle Workflow for Agent Deactivation**

> **Note:** Full Lifecycle Workflows requires Microsoft Entra ID Governance add-on for automated triggers. With P2 you can configure the workflow — you may not be able to trigger it automatically on a schedule without the Governance add-on. You can trigger it manually for lab purposes.

1. Entra admin centre → **Identity governance** → **Lifecycle workflows**
2. Click **Create workflow**
3. Template: select **Custom** (no pre-built template for agent deactivation)
4. Name: `AI-Agent-Deactivation-Workflow`
5. Description: `Deactivate AI agent identity when sponsor confirms retirement`
6. Trigger: **On-demand** (manual for lab)
7. Add tasks:
   - Task 1: **Disable account** — disables the service principal sign-in
   - Task 2: **Send email to manager** — notifies the sponsor of deactivation
8. Save workflow

**Step 5 — Run the Deactivation Workflow Manually (Simulation)**

> Do not actually run this against your `CustomerSupportAgent-Instance-001` or you will disable it and need to re-enable it for Lab 10. Instead run it against a throwaway test App Registration.

1. Create a test App Registration: `test-agent-to-deactivate`
2. Go to your Lifecycle Workflow → click **Run on demand**
3. Select the test App Registration as the target
4. Run the workflow
5. Verify the test App Registration is now disabled in Entra ID

**Step 6 — Review the Audit Log**

1. Entra admin centre → **Monitoring** → **Audit logs**
2. Filter: Service = **Lifecycle workflows**
3. You will see the workflow execution with every task logged — who initiated it, what was done, when

---

### What You Learned

- How to run Access Reviews on Azure resource role assignments for service principals
- The governance principle: non-response = deny access for AI agents (not keep)
- How to design a Lifecycle Workflow for agent deactivation
- The audit trail from Access Reviews and Lifecycle Workflows that supports regulated environment compliance

### Interview Talking Point After This Lab
*"AI agent governance follows the same lifecycle model as human identity — create with a sponsor, govern with access reviews, deactivate when no longer needed. I've configured access reviews for agent role assignments with auto-deny on non-response — an agent whose sponsor does not re-certify quarterly automatically loses its role assignments. Lifecycle Workflows handle the deactivation sequence: disable the service principal, notify the sponsor, remove role assignments. The full lifecycle is auditable from day one."*

---

## Lab 10 — Monitor and Audit AI Agent Sign-In Activity in Sentinel

### What You Will Learn
- How to stream agent identity sign-in logs into Log Analytics
- How to write a KQL query that surfaces anomalous agent authentication patterns
- How to create a Sentinel alert that fires when an agent authenticates from an unexpected context

### Why This Matters for BoC
BoC needs to know what AI agents are doing with their access. This lab closes the monitoring loop — not just creating and governing agent identities, but actively detecting when something looks wrong.

### Reference Learning
- 📖 Microsoft Learn: [Connect Entra ID to Microsoft Sentinel](https://learn.microsoft.com/en-us/azure/sentinel/connect-azure-active-directory)
- 📖 Microsoft Learn: [KQL quick reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- 🎥 YouTube: [Microsoft Sentinel KQL for beginners — Rod Trent](https://www.youtube.com/watch?v=IKCCe_ZCLaM)

---

### Step-by-Step Instructions

**Step 1 — Set Up Log Analytics Workspace**

1. portal.azure.com → search **Log Analytics workspaces** → **Create**
2. Resource group: `rg-iam-labs`
3. Name: `law-iamlabs`
4. Region: Canada Central
5. Click **Review + create** → **Create**

**Step 2 — Connect Microsoft Sentinel to the Workspace**

1. Search **Microsoft Sentinel** → **Create**
2. Select `law-iamlabs` workspace → **Add**
3. Sentinel is now active on your workspace

**Step 3 — Connect Entra ID Sign-In Logs to Sentinel**

1. In Microsoft Sentinel → **Content management** → **Content hub**
2. Search for **Microsoft Entra ID** → Install
3. Go to **Configuration** → **Data connectors**
4. Find **Microsoft Entra ID** → click it → **Open connector page**
5. Under **Configuration** check:
   - ✅ Sign-in logs
   - ✅ Audit logs
   - ✅ Service principal sign-in logs ← this is the one that captures agent sign-ins
6. Click **Apply changes**

> **Note:** It takes 15-30 minutes for logs to start flowing. Continue with the KQL steps while you wait.

**Step 4 — Write KQL Queries for Agent Activity**

Once logs are flowing, go to Sentinel → **Logs** and run these queries:

**Query 1 — All service principal sign-ins in last 24 hours:**
```kql
AADServicePrincipalSignInLogs
| where TimeGenerated > ago(24h)
| project TimeGenerated, ServicePrincipalName, IPAddress, Location, ResultType, ResultDescription
| order by TimeGenerated desc
```

**Query 2 — Find agent sign-ins from unexpected locations:**
```kql
AADServicePrincipalSignInLogs
| where TimeGenerated > ago(7d)
| where ServicePrincipalName contains "CustomerSupportAgent"
| extend LocationStr = strcat(Location, " / ", IPAddress)
| summarize SignInCount = count(), Locations = make_set(LocationStr) by ServicePrincipalName, bin(TimeGenerated, 1h)
| where array_length(Locations) > 1  // Multiple locations in same hour = suspicious
| order by TimeGenerated desc
```

**Query 3 — Failed agent authentications (potential credential stuffing):**
```kql
AADServicePrincipalSignInLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"  // 0 = success, anything else = failure
| summarize FailureCount = count(), FailureReasons = make_set(ResultDescription)
    by ServicePrincipalName, IPAddress
| where FailureCount > 5  // More than 5 failures from same IP = alert
| order by FailureCount desc
```

**Query 4 — Agent signing in outside business hours:**
```kql
AADServicePrincipalSignInLogs
| where TimeGenerated > ago(7d)
| where ServicePrincipalName contains "CustomerSupportAgent"
| extend HourOfDay = datetime_part("Hour", TimeGenerated)
| where HourOfDay < 6 or HourOfDay > 22  // Outside 6am-10pm
| project TimeGenerated, ServicePrincipalName, IPAddress, HourOfDay, ResultType
| order by TimeGenerated desc
```

**Step 5 — Create a Sentinel Analytics Rule (Alert)**

1. Sentinel → **Configuration** → **Analytics**
2. Click **Create** → **Scheduled query rule**
3. Name: `AI-Agent-Unexpected-Location`
4. Description: `Fires when an AI agent service principal authenticates from outside Azure infrastructure IPs`
5. Tactics: **Initial Access**
6. Severity: **High**
7. Query:
```kql
AADServicePrincipalSignInLogs
| where ServicePrincipalName contains "CustomerSupportAgent"
| where IPAddress !startswith "20."  // Not Azure Canada Central IP range
| where ResultType == "0"  // Successful sign-in — this is the dangerous scenario
| project TimeGenerated, ServicePrincipalName, IPAddress, Location
```
8. Run query every: **5 minutes**
9. Lookup data from last: **5 minutes**
10. Alert threshold: generate alert when number of query results is **greater than 0**
11. Click **Next: Incident settings** → enable incident creation
12. Click **Review + create** → **Create**

**Step 6 — Review the Full Detection and Response Chain**

Write out the chain you just built — this is your answer to any "walk me through your monitoring" question:

```
AI Agent signs in
  └─ Sign-in event logged in Entra ID
       └─ Log streamed to Log Analytics via Sentinel data connector
            └─ Sentinel Analytics rule evaluates every 5 minutes
                 └─ If agent signs in from non-Azure IP:
                      └─ Sentinel alert fires → Incident created
                           └─ Security team investigates
                                └─ If confirmed anomalous:
                                     └─ Disable service principal in Entra ID
                                          └─ CAE revokes active tokens immediately
```

---

### What You Learned

- How to stream Entra ID service principal sign-in logs into Microsoft Sentinel
- Four KQL queries for detecting anomalous AI agent behaviour — location anomalies, failed auth spikes, off-hours activity
- How to create an analytics rule that generates an alert when an agent authenticates from an unexpected location
- The full detection-to-response chain that closes the monitoring loop on AI agent identity

### Interview Talking Point After This Lab
*"AI agent monitoring follows the same pattern as human identity monitoring but scoped to service principal sign-in logs. I've configured Sentinel to stream Entra ID service principal logs and written KQL rules that detect unexpected agent behaviour — authentication from non-Azure IPs, failed auth spikes, off-hours sign-ins. An analytics rule fires automatically when the agent authenticates from outside the expected network context. The response chain ends with disabling the service principal in Entra ID which triggers immediate CAE token revocation."*

---

## Lab Completion Summary

Once you have completed all 10 labs, you can say the following with full accuracy:

### CI/CD — What You Can Now Own

| Claim | Evidence from Labs |
|---|---|
| Created Azure DevOps pipelines with YAML | Lab 1 |
| Configured Workload Identity Federation service connections | Lab 2 |
| Pulled Key Vault secrets in pipelines — no stored credentials | Lab 3 |
| Deployed Entra ID App Registrations via pipeline (IAM as code) | Lab 4 |
| Governed pipeline identities with Access Reviews and CA policies | Lab 5 |

### AI Agent Identity — What You Can Now Own

| Claim | Evidence from Labs |
|---|---|
| Registered agent identities in Entra Agent ID | Lab 6 |
| Scoped OAuth2 permissions using least-privilege principles | Lab 7 |
| Applied Conditional Access to workload identities (agent-specific) | Lab 8 |
| Governed agent lifecycle with Access Reviews and Lifecycle Workflows | Lab 9 |
| Monitored agent sign-in activity with KQL and Sentinel alerts | Lab 10 |

---

## Clean-Up Checklist

After completing all labs delete these resources to avoid any ongoing cost:

- [ ] Resource group `rg-iam-labs` (deletes Key Vault, Log Analytics workspace)
- [ ] App Registration: `lab4-demo-app-dev`
- [ ] App Registration: `test-agent-to-deactivate`
- [ ] Agent identity: `CustomerSupportAgent-Instance-001`
- [ ] Agent blueprint: `CustomerSupportAgent-Blueprint`
- [ ] Sentinel workspace (billed per GB ingested — delete after labs)
- [ ] Azure DevOps project: `IAM-Pipeline-Labs` (optional — free to keep)

> **Keep:** The Azure DevOps organisation and service connection — these are free and useful for future lab work.

---

## What to Say in Your Next Interview

The honest framing after completing these labs:

*"Since my last interview I have completed hands-on labs covering both areas. For CI/CD, I built Azure DevOps pipelines from scratch, configured Workload Identity Federation service connections with no stored secrets, pulled Key Vault secrets at runtime, and deployed App Registrations via pipeline as code. For AI agent identity, I registered agent identities in Entra Agent ID, scoped OAuth2 permissions using least-privilege principles, applied Conditional Access policies scoped to those service principals, governed the agent lifecycle with Access Reviews, and built Sentinel KQL rules to detect anomalous agent authentication patterns. I now have hands-on experience across both areas rather than just conceptual understanding."*

---

*End of Lab Document*

---
**Kamran Arif | IAM Engineer | SC-300 | AZ-305 | AZ-104**
**kamranarif.ca@outlook.com | linkedin.com/in/karifa | 905-906-2786**
