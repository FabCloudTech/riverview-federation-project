# Riverview Regional Medical Center: Federation, OAuth 2.0 & OIDC Lab

**Environment:** Microsoft Entra ID | Salesforce Developer Edition | Microsoft Graph Explorer | PowerShell 7.6  
**Simulated Organization:** Riverview Regional Medical Center  
**Protocols:** SAML 2.0 | OAuth 2.0 | OpenID Connect (OIDC)  
**Framework Mapping:** NIST SP 800-53 Rev. 5  

---

## Overview

This project is the federation and authentication protocol capstone of the Riverview Regional Medical Center IAM program. It covers three distinct identity integration patterns that every enterprise environment uses: SAML 2.0 for legacy application SSO, OAuth 2.0 for delegated API authorization, and OIDC for modern identity-aware SSO.

All three protocols were implemented in a live Entra ID tenant and tested end-to-end. The project concludes with a PowerShell automation script that programmatically replicates the full app registration workflow, and a JWT claims analysis that maps token behavior to governance controls.

---

## Business Problem

Enterprise environments run multiple authentication protocols simultaneously. Salesforce authenticates via SAML. Internal APIs use OAuth tokens. Modern apps layer OIDC on top for user identity. An IAM team that only understands one protocol creates integration gaps, misconfigurations, and ungoverned machine credentials.

This project demonstrates the ability to implement, validate, and govern all three protocols in a single tenant environment: and document the security implications of each.

---

## Part A: SAML 2.0 Federation with Salesforce

Configured SAML SSO between Entra ID (Identity Provider) and Salesforce Developer Edition (Service Provider). A user authenticates against Entra ID and Salesforce accepts the signed SAML assertion without requiring a separate Salesforce password.

**What was configured:**

| Setting | Value |
|---|---|
| Identity Provider | Microsoft Entra ID |
| Service Provider | Salesforce Developer Edition |
| Entity ID | Salesforce tenant Entity ID |
| Reply URL (ACS) | Salesforce ACS URL |
| Signing Certificate | Downloaded from Entra, uploaded to Salesforce |
| Claims Mapping | UPN mapped to Salesforce username |

**Validated:** Federated login confirmed. Salesforce authenticated the user via Entra ID assertion with no separate Salesforce credential required.

**NIST Mapping:** IA-8 (Identification and Authentication: Non-Organizational Users), SC-8 (Transmission Confidentiality)

---

## Part B: OAuth 2.0 / OIDC via Microsoft Graph

Registered an application in Entra ID and used it to make authenticated API calls to Microsoft Graph, demonstrating the OAuth 2.0 delegated authorization flow and OIDC identity layer in action.

### App Registration: Manual

**App name:** RVR-OAuth-OIDC-Lab  
**Delegated permissions granted:**

| Permission | Purpose |
|---|---|
| openid | OIDC authentication: establishes who the user is |
| profile | Returns basic profile claims |
| email | Returns user email address |
| User.Read | Read signed-in user profile |
| User.Read.All | Read all users in the tenant |

### Graph Explorer Queries

Three live API calls were made to validate the OAuth token and demonstrate delegated access:

```
GET https://graph.microsoft.com/v1.0/me
```
Returns the authenticated user's profile: proves OIDC identity layer is working.

```
GET https://graph.microsoft.com/v1.0/me/memberOf
```
Returns group memberships for the signed-in user: demonstrates scoped delegated access.

```
GET https://graph.microsoft.com/v1.0/users
```
Returns all provisioned users in the tenant: validates User.Read.All permission grant.

**NIST Mapping:** AC-4 (Information Flow), IA-5 (Authenticator Management), AC-17 (Remote Access)

---

## Part C: JWT Claims Analysis

The access token issued by Entra ID during the OAuth flow was decoded on jwt.ms. Every claim maps to a governance control.

| Claim | Meaning | Governance Significance |
|---|---|---|
| `aud` | Audience: intended recipient app | App must reject tokens where aud does not match its own client ID. Prevents token relay attacks. |
| `iss` | Issuer: Entra ID tenant STS URL | Identifies which tenant issued the token. Used to restrict sign-in to authorized tenants only. |
| `iat` | Issued At timestamp | Tokens should not be accepted if iat is significantly in the past. |
| `nbf` | Not Before timestamp | Replay protection. Token cannot be used before this time. |
| `exp` | Expiration timestamp | Defines token lifetime. Machine tokens with no expiration are a critical NHI governance risk. |
| `acr` | Authentication Context Class | Value 0 = auth did not meet ISO/IEC 29115. Value 1 = compliant. Monitor for acr=0 on sensitive resources. |
| `acrs` | Authentication Context Reference | `pfdr` = Continuous Access Evaluation enabled. Token can be revoked proactively on risk signals. |

**Key governance finding:** Any machine identity token without an `exp` claim or with expiration set years in the future is a high-severity finding in an NHI access review. This is the exact pattern behind the Okta 2023 and Snowflake 2024 breaches.

---

## Part D: PowerShell Automation

After manually registering the app through the portal, the same workflow was automated using PowerShell and the Microsoft Graph SDK. This demonstrates both the governance understanding (manual) and the engineering execution (automated).

**Script:** `register-oauth-app.ps1`

**What the script does:**
- Creates the app registration in Entra ID
- Adds all 4 delegated API permissions programmatically
- Generates a client secret with a 1-year expiration
- Exports the app configuration to CSV for documentation

```powershell
Connect-MgGraph -Scopes "Application.ReadWrite.All"

$app = New-MgApplication -DisplayName "RVR-OAuth-OIDC-Lab-Auto" `
    -SignInAudience "AzureADMyOrg"

$graphResourceId = "00000003-0000-0000-c000-000000000000"

$permissions = @(
    "64a6cdd6-aab1-4aad-94b8-3cc8405e90d0", # email
    "37f7f235-527c-4136-accd-4a02d197296e", # openid
    "14dad69e-099b-42c9-810b-d002981feec1", # profile
    "e1fe6dd8-ba31-4d61-89e7-88639da4683d"  # User.Read
)

$requiredAccess = @{
    ResourceAppId  = $graphResourceId
    ResourceAccess = $permissions | ForEach-Object {
        @{ Id = $_; Type = "Scope" }
    }
}

Update-MgApplication -ApplicationId $app.Id `
    -RequiredResourceAccess @($requiredAccess)

$secret = Add-MgApplicationPassword -ApplicationId $app.Id `
    -PasswordCredential @{
        DisplayName = "RVR-Lab-Secret"
        EndDateTime = (Get-Date).AddYears(1)
    }

[PSCustomObject]@{
    AppName    = $app.DisplayName
    ClientId   = $app.AppId
    TenantId   = (Get-MgContext).TenantId
    SecretHint = $secret.Hint
    Created    = Get-Date -Format "yyyy-MM-dd"
} | Export-Csv -Path "RVR_AppReg_Config.csv" -NoTypeInformation
```

**Result:** Both apps confirmed live in Entra App Registrations: one manually created, one script-created: each with active client secrets.

---

## Protocol Comparison

| Protocol | Function | Token Format | Use Case |
|---|---|---|---|
| SAML 2.0 | Federated SSO via signed XML assertion | XML | Legacy enterprise SaaS: Salesforce, ServiceNow, on-prem apps |
| OAuth 2.0 | Delegated authorization: scoped resource access | JWT Access Token | API access, mobile apps, modern SaaS integrations |
| OIDC | Identity layer on top of OAuth | JWT ID Token + Access Token | Modern SSO: any app that needs to know who the user is |

**Architecture note:** SAML was used for Salesforce because Salesforce's SSO is SAML-native. OAuth 2.0 and OIDC were used for Microsoft Graph because it is a modern API expecting token-based authorization. In a mature hybrid enterprise, both protocols coexist: SAML for legacy federation, OIDC/OAuth for cloud-native API access.

---

## NIST SP 800-53 Control Mapping

| Control | Name | Implementation |
|---|---|---|
| AC-4 | Information Flow | OAuth scopes restrict token access to approved resources only |
| AC-17 | Remote Access | Federated and token-based auth enforces controlled remote access |
| IA-5 | Authenticator Management | Client secrets scoped with expiration; token lifetime enforced via exp claim |
| IA-8 | Non-Org User ID and Auth | SAML federation authenticates external Salesforce users via Entra assertion |
| SC-8 | Transmission Confidentiality | TLS enforced on all token exchange and SAML assertion transmission |

---

## Findings

**Finding: No NHI Governance Policy for App Registrations**

Two app registrations were created during this lab with no formal ownership, rotation schedule, or decommissioning process. In a production environment, this is a governance gap. Ungoverned app registrations with active client secrets are a known attack vector: service principal credentials with no owner and no rotation are the machine identity equivalent of a stale privileged account.

Recommendation: Apply a formal NHI governance policy to all app registrations covering ownership assignment, secret rotation cadence (90 days), access review frequency (quarterly), and decommissioning procedures when the app is no longer needed.

See companion project: [NHI Lifecycle Automation Engine](https://github.com/FabCloudTech) *(coming soon)*

---

## Companion Project

**Days 1-6: Identity Lifecycle Management Governance Program**  
Full JML lifecycle, PIM, access reviews, app assignment, offboarding, and Zero Trust CA policies.  
See: [entra-id-iam-project](https://github.com/FabCloudTech/entra-id-iam-project)

---

## Skills Demonstrated

SAML 2.0 Federation | OAuth 2.0 | OpenID Connect | JWT Analysis | PowerShell Automation | Microsoft Graph SDK | App Registration Governance | Token Lifecycle Management | Non-Human Identity Risk | NIST SP 800-53 | Entra ID Administration

---

*Fabella Terry | IAM Governance Analyst | [fabcloudtech.github.io](https://fabcloudtech.github.io)*
