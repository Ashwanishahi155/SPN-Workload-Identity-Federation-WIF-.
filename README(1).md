# 🚀 Azure DevOps → Azure Service Connection using SPN + Workload Identity Federation (WIF)

> **Portal screenshots + click-by-click practice guide for Azure DevOps,
> Microsoft Entra ID, Azure RBAC, Terraform and AKS**

This README documents the complete **Azure DevOps → Azure authentication
→ Terraform → AKS** flow using an **App Registration / Service Principal
(`AKS-SPN`)** with **Workload Identity Federation (WIF)**.

The workflow covered in the source presentation is:

``` text
Microsoft Entra App Registration
        │
        │ Client ID + Tenant ID
        ▼
Azure DevOps ARM Service Connection
        │
        │ WIF → Issuer + Subject
        ▼
Federated Credential
        │
        │ Trust established
        ▼
Azure Subscription RBAC
        │
        │ Contributor
        ▼
ADO Pipeline
        │
        ├── AzureCLI → az account show
        ├── Terraform → init → validate → plan → apply
        ▼
AKS Deployment
        │
        ├── ACR
        ├── VNet / Subnet
        ├── Log Analytics
        └── Kubernetes workloads
```

------------------------------------------------------------------------

## 📌 What this project teaches

By completing this lab, you practice:

-   Microsoft Entra ID App Registration
-   Service Principal / SPN concept
-   Azure DevOps Azure Resource Manager Service Connection
-   Workload Identity Federation (WIF)
-   Federated Credentials
-   Azure Subscription-level RBAC
-   AzureCLI pipeline authentication
-   Terraform CI/CD
-   AKS deployment flow
-   Troubleshooting `AuthorizationFailed`
-   End-to-end Azure authentication without relying on a client secret
    in the WIF flow

------------------------------------------------------------------------

# 🏗️ End-to-End Architecture

``` text
┌──────────────────────────────┐
│ Microsoft Entra ID           │
│ App Registration: AKS-SPN    │
│                              │
│ Client ID                    │
│ Tenant ID                    │
└──────────────┬───────────────┘
               │
               │ Trust
               ▼
┌──────────────────────────────┐
│ Azure DevOps                 │
│ ARM Service Connection       │
│                              │
│ Workload Identity Federation │
│ Issuer + Subject Identifier  │
└──────────────┬───────────────┘
               │
               │ Federated Credential
               ▼
┌──────────────────────────────┐
│ Microsoft Entra ID           │
│ Federated Credential         │
│ Audience:                    │
│ api://AzureADTokenExchange   │
└──────────────┬───────────────┘
               │
               │ Azure RBAC
               ▼
┌──────────────────────────────┐
│ Azure Subscription           │
│ Role: Contributor            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Azure DevOps Pipeline        │
│                              │
│ AzureCLI                     │
│ Terraform                    │
│ AKS Deployment               │
└──────────────────────────────┘
```

------------------------------------------------------------------------

# 🔐 Step 1 --- Create / Use `AKS-SPN`

Go to:

**Azure Portal → Microsoft Entra ID → App registrations → AKS-SPN →
Overview**

Collect:

  Field                         Use
  ----------------------------- ------------------------------------------------
  **Application (client) ID**   Client ID for ADO
  **Directory (tenant) ID**     Tenant ID for ADO
  **Object ID**                 Different value --- do not use it as Client ID

The source presentation explicitly distinguishes the **Application
(client) ID**, **Directory (tenant) ID**, and **Object ID**.
fileciteturn1file0L12-L17

### 📸 Portal screenshot

![AKS-SPN App Registration Overview](images/portal-step-1.png)

------------------------------------------------------------------------

# 🔗 Step 2 --- Create Azure DevOps Service Connection

Go to:

**Azure DevOps → Project Settings → Service connections → New service
connection**

Select:

**Azure Resource Manager**

Then click **Next**.

The presentation uses this exact portal path and connection type.
fileciteturn1file0L21-L25

### 📸 Portal screenshot

![Azure DevOps New Service Connection](images/portal-step-2.png)

------------------------------------------------------------------------

# ⚙️ Step 3 --- Select Subscription Scope

On the Azure Resource Manager service connection screen:

-   **Scope Level:** `Subscription`
-   Provide/select the required subscription details.
-   Keep your service connection name consistent.

Example:

``` text
Service Connection Name:
sc-aks-terraform
```

If Azure does not automatically populate the subscription, use the
**manual / WIF** path rather than forcing the automatic subscription
flow. fileciteturn1file0L29-L35

### 📸 Portal screenshot

![Azure Resource Manager Service Connection](images/portal-step-3.png)

------------------------------------------------------------------------

# 🔑 Step 4 --- Configure Workload Identity Federation

In the manual ARM service connection flow, Azure DevOps provides:

-   **Issuer**
-   **Subject identifier**

Copy both values **exactly**.

These values establish the trust relationship that will later be
configured under the `AKS-SPN` App Registration's federated credentials.
fileciteturn1file0L37-L41

### 📸 Portal screenshot

![Azure DevOps WIF Issuer and Subject](images/portal-step-4.png)

> ⚠️ **Important:** Do not manually change the Issuer or Subject values.
> Copy the generated values exactly.

------------------------------------------------------------------------

# 🧾 Step 5 --- Enter Subscription + Identity Details

Use:

``` text
Scope Level       → Subscription
Subscription ID   → <YOUR-SUBSCRIPTION-ID>
Subscription Name → <YOUR-SUBSCRIPTION-NAME>
Client ID         → <AKS-SPN-APPLICATION-CLIENT-ID>
Tenant ID         → <YOUR-DIRECTORY-TENANT-ID>
```

The source guide specifically calls out Subscription ID, Subscription
name, Application (client) ID and Directory (tenant) ID for this flow.
fileciteturn1file0L45-L50

### 📸 Portal screenshot

------------------------------------------------------------------------

# 🔐 Step 6 --- Add Federated Credential in Azure

Go to:

**Azure Portal → Microsoft Entra ID → App registrations → AKS-SPN**

Then:

**Certificates & secrets → Federated credentials → Add credential**

Configure:

  Field                Value
  -------------------- ------------------------------
  Issuer               **ADO-generated Issuer**
  Subject identifier   **ADO-generated Subject**
  Audience             `api://AzureADTokenExchange`
  Name                 `ado-terraform-wif`

The presentation specifies the `Other issuer` / manual federation flow,
with the ADO Issuer and Subject and the audience
`api://AzureADTokenExchange`. fileciteturn1file0L54-L61

### 📸 Portal screenshot

------------------------------------------------------------------------

# 🛡️ Step 7 --- Give Azure Subscription RBAC

Go to:

**Azure Portal → Subscriptions → Your Subscription → Access control
(IAM)**

Then:

1.  **Add**
2.  **Add role assignment**
3.  Role → **Contributor**
4.  Click **Next**
5.  **Assign access to:** User, group, or service principal
6.  **Select members**
7.  Search for `AKS-SPN`
8.  Select the correct service principal
9.  **Next**
10. **Review + assign**

This subscription-level RBAC step is part of the documented flow.
fileciteturn1file0L64-L70

### ⚠️ Why this matters

WIF establishes **authentication/trust**, but Azure RBAC determines
**what the identity is allowed to do** in the subscription.

So remember:

``` text
WIF  = Trust / Authentication
RBAC = Authorization / Permissions
```

------------------------------------------------------------------------

# ✅ Step 8 --- Verify & Save Service Connection

Return to:

**Azure DevOps → Project Settings → Service connections**

Open the WIF service connection.

Click:

**Finish setup → Verify and save**

Expected result:

``` text
Verification successful
```

If you get:

``` text
Microsoft.Resources/subscriptions/read
AuthorizationFailed
```

check the subscription-level RBAC assignment and confirm that the
**exact service principal identity** received the role.
fileciteturn1file0L73-L78

------------------------------------------------------------------------

# 🧪 Step 9 --- First Pipeline Test

Before adding Terraform, first prove that Azure DevOps can authenticate
to Azure.

Example:

``` yaml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:
  - task: AzureCLI@2
    displayName: "Test Azure Connection"
    inputs:
      azureSubscription: 'sc-aks-terraform'
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        echo "Azure authentication successful"
        az account show
```

The expected validation is that `az account show` returns the expected
Azure subscription. fileciteturn1file0L81-L84

### 🎯 Interview-ready explanation

> **"First I validate the Azure DevOps service connection using AzureCLI
> and `az account show`. Once authentication is confirmed, I integrate
> Terraform with the same WIF identity."**

------------------------------------------------------------------------

# 🏗️ Step 10 --- Terraform Pipeline

Recommended Terraform sequence:

``` text
fmt
  ↓
init
  ↓
validate
  ↓
plan
  ↓
apply
```

Example conceptual pipeline:

``` yaml
steps:
  - script: terraform fmt -check
    displayName: "Terraform Format Check"

  - script: terraform init
    displayName: "Terraform Init"

  - script: terraform validate
    displayName: "Terraform Validate"

  - script: terraform plan
    displayName: "Terraform Plan"

  - script: terraform apply -auto-approve
    displayName: "Terraform Apply"
```

Keep the Azure DevOps service connection name consistent with the
`azureSubscription` value used by the pipeline. After AzureCLI
authentication works, configure the Terraform AzureRM provider for the
same WIF identity. fileciteturn1file0L87-L97

> 💡 **Important:** Successful `az account show` proves the AzureCLI
> task can authenticate. Terraform AzureRM authentication still needs to
> be configured correctly for the same identity.

------------------------------------------------------------------------

# ☸️ Step 11 --- AKS Deployment

The final infrastructure/application flow is:

``` text
Terraform
   │
   ├── Resource Group
   ├── VNet
   ├── Subnet
   ├── ACR
   ├── Log Analytics
   └── AKS
          │
          ▼
       kubectl
          │
          ▼
   Kubernetes Workloads
```

The presentation specifically lists Resource Group, VNet/Subnet, ACR,
Log Analytics and AKS before moving to `kubectl`.
fileciteturn1file0L100-L109

After AKS is ready, practice:

-   Deployment
-   Service
-   Ingress
-   HPA
-   Readiness/Liveness probes
-   ConfigMap
-   Secret
-   Troubleshooting
-   Helm where appropriate

------------------------------------------------------------------------

# 🔄 Complete Flow --- One Page

``` text
1. Create / use AKS-SPN
        ↓
2. Get Client ID + Tenant ID
        ↓
3. Azure DevOps → Service Connections
        ↓
4. Azure Resource Manager
        ↓
5. Manual / WIF
        ↓
6. Copy Issuer + Subject
        ↓
7. Azure → AKS-SPN → Federated credentials
        ↓
8. Add Issuer + Subject + Audience
        ↓
9. Subscription → IAM
        ↓
10. Contributor → AKS-SPN
        ↓
11. ADO → Verify & Save
        ↓
12. AzureCLI → az account show
        ↓
13. Terraform
        ↓
14. AKS
        ↓
15. kubectl / Kubernetes deployment
```

------------------------------------------------------------------------

# 🧠 WIF vs Client Secret

For this lab, the important concept is:

``` text
Traditional authentication:
ADO → Client ID + Client Secret → Azure

WIF:
ADO → Federated Token → Microsoft Entra ID → Azure
```

### Why WIF is useful

-   No long-lived client secret is required for the WIF trust flow.
-   Trust is established using the federated credential.
-   Azure RBAC controls permissions after authentication.
-   It fits CI/CD authentication scenarios.

------------------------------------------------------------------------

# 🛠️ Troubleshooting

## ❌ Error: `AuthorizationFailed`

Typical message:

``` text
Microsoft.Resources/subscriptions/read
```

### Check:

``` text
Azure Subscription
    ↓
Access control (IAM)
    ↓
Role assignments
    ↓
AKS-SPN
```

Make sure the intended service principal has the required role at the
required scope.

The source guide specifically associates this error with checking
subscription-level RBAC and the exact service principal identity.
fileciteturn1file0L73-L78

------------------------------------------------------------------------

## ❌ Client ID / Object ID confusion

Remember:

``` text
Application (client) ID → Client ID
Directory (tenant) ID   → Tenant ID
Object ID               → Different identifier
```

Do **not** paste Object ID where ADO asks for Application (client) ID.
fileciteturn1file0L12-L17

------------------------------------------------------------------------

## ❌ Subscription does not appear

If the automatic ADO subscription flow says that no active Azure
subscription is available:

``` text
Don't force the automatic flow.

Use:
Azure Resource Manager
        ↓
Manual / App registration or managed identity
        ↓
Workload Identity Federation
```

The captured portal screen in the guide shows the
subscription-not-available condition and the documented workaround is to
use the manual/WIF path. fileciteturn1file0L29-L35

------------------------------------------------------------------------

# 🔒 Security Checklist

Before using this in a real environment:

-   [ ] Avoid committing secrets to Git.
-   [ ] Do not put client secrets in YAML.
-   [ ] Use WIF where supported.
-   [ ] Give the service principal only the permissions it actually
    needs.
-   [ ] Prefer narrower RBAC scope than subscription-wide access where
    practical.
-   [ ] Keep Terraform state secure.
-   [ ] Protect production `apply`.
-   [ ] Use approvals/environment controls for production.
-   [ ] Keep service connection permissions restricted to required
    pipelines.

------------------------------------------------------------------------

# 🎤 Interview Explanation --- 60 Seconds

> **"For Azure DevOps authentication, I created an Entra ID App
> Registration called AKS-SPN. I use its Application Client ID and
> Tenant ID in an Azure Resource Manager service connection. Instead of
> using a long-lived client secret, I use Workload Identity Federation.
> Azure DevOps generates an Issuer and Subject identifier, which I
> configure as a Federated Credential on the AKS-SPN application with
> the Azure AD token exchange audience. Then I assign the required Azure
> RBAC role to the service principal at the subscription scope. After
> that, I verify the connection using an AzureCLI task with
> `az account show`. Once authentication is validated, I configure
> Terraform with the same WIF identity and run fmt, init, validate, plan
> and apply. Finally, Terraform provisions the AKS infrastructure and I
> deploy workloads using kubectl or Helm."**

------------------------------------------------------------------------

# 📋 Quick Reference Table

  ---------------------------------------------------------------------
  Component                          Purpose
  ---------------------------------- ----------------------------------
  **AKS-SPN**                        Microsoft Entra
                                     application/service identity

  **Client ID**                      Identifies the application

  **Tenant ID**                      Identifies the Entra tenant

  **ADO ARM Service Connection**     Connects Azure DevOps pipeline to
                                     Azure

  **WIF**                            Establishes federated trust

  **Issuer**                         Identifies the token issuer

  **Subject**                        Identifies the federated workload

  **Federated Credential**           Defines trusted external identity

  **Audience**                       `api://AzureADTokenExchange`

  **RBAC**                           Controls Azure permissions

  **Contributor**                    Lab role used in this guide

  **AzureCLI**                       First authentication test

  **Terraform**                      Infrastructure automation

  **AKS**                            Managed Kubernetes platform

  **ACR**                            Container image registry

  **Log Analytics**                  Log/monitoring workspace
  ---------------------------------------------------------------------

------------------------------------------------------------------------

# 🎯 Practice Checklist

### Azure / Entra

-   [ ] Create App Registration
-   [ ] Identify Client ID
-   [ ] Identify Tenant ID
-   [ ] Understand Object ID
-   [ ] Create Federated Credential
-   [ ] Understand WIF

### Azure DevOps

-   [ ] Create ARM Service Connection
-   [ ] Configure manual/WIF flow
-   [ ] Copy Issuer
-   [ ] Copy Subject
-   [ ] Verify service connection

### Azure RBAC

-   [ ] Open Subscription IAM
-   [ ] Add role assignment
-   [ ] Select AKS-SPN
-   [ ] Validate permissions

### Pipeline

-   [ ] Run `az account show`
-   [ ] Run Terraform `fmt`
-   [ ] Run `terraform init`
-   [ ] Run `terraform validate`
-   [ ] Run `terraform plan`
-   [ ] Run `terraform apply`

### AKS

-   [ ] Create AKS using Terraform
-   [ ] Push image to ACR
-   [ ] Connect with kubectl
-   [ ] Deploy application
-   [ ] Configure Service
-   [ ] Configure Ingress
-   [ ] Configure HPA
-   [ ] Practice troubleshooting

------------------------------------------------------------------------

# ⭐ Final Cheat Sheet

``` text
AKS-SPN
   ↓
Client ID + Tenant ID
   ↓
ADO ARM Service Connection
   ↓
WIF
   ↓
Issuer + Subject
   ↓
Federated Credential
   ↓
Contributor / Required RBAC
   ↓
Verify & Save
   ↓
AzureCLI
   ↓
Terraform
   ↓
AKS
   ↓
kubectl / Helm
```

**Core concept to remember:**

> 🔐 **WIF establishes trust; RBAC grants permissions; Azure DevOps uses
> the service connection to access Azure; Terraform provisions
> infrastructure; AKS runs the workloads.**

------------------------------------------------------------------------

## 📚 Source Presentation

This README was created from the uploaded **Azure DevOps → Azure Service
Connection --- SPN + Workload Identity Federation (WIF) --- Portal
Screenshots + Click-by-Click Practice Guide** presentation, including
its portal screenshots and step sequence. The presentation's final cheat
sheet summarizes the same Azure-side App Registration/Federated
Credential/RBAC flow, ADO-side ARM + WIF service connection,
`az account show` validation, and Terraform → AKS deployment path.
fileciteturn1file0L113-L127
