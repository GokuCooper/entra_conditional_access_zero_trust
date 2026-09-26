# Conditional Access policy matrix

Tenant: DKN IAM Workforce Lab (dknorringtonzohomail.onmicrosoft.com)
License: Microsoft Entra ID P2 (trial), assigned through the group LIC_Entra_ID_P2_Users

| Policy | Users included | Users excluded | Target resources | Condition | Grant | Final state |
| --- | --- | --- | --- | --- | --- | --- |
| CA001_Require_MFA_All_Users | All users | CA_Exclude_BreakGlass | All resources | None | Require multifactor authentication | On |
| CA002_Require_Compliant_Device | All users | CA_Exclude_BreakGlass | All resources | None (all platforms kept in scope) | Require device to be marked as compliant | Report only (no Intune in this tenant) |
| CA003_Block_High_Risk_SignIns | All users | CA_Exclude_BreakGlass | All resources | Sign in risk: High | Block access | On |
| CA004_Block_Outside_Allowed_Countries | All users | CA_Exclude_BreakGlass, CA_Exclude_Travel_Approved | All resources | Network: include any location, exclude NL_Allowed_Countries | Block access | On |
| CA005 break glass alert | Not a Conditional Access policy | | | | | Deferred to Week 8 (PowerShell and Microsoft Graph). Interim control: saved sign in log filter |

## Supporting objects

| Object | Type | Purpose |
| --- | --- | --- |
| breakglass01 | Cloud only user, Global Administrator, onmicrosoft.com UPN | Emergency access if Conditional Access ever locks admins out |
| CA_Exclude_BreakGlass | Security group, assigned | The only way an account is excluded from every policy |
| CA_Exclude_Travel_Approved | Security group, assigned | Temporary travel exceptions for CA004 only |
| LIC_Entra_ID_P2_Users | Security group, assigned | Group based licensing for Entra ID P2 |
| NL_Allowed_Countries | Countries named location (IP based), United States, unknown countries not included | The "safe" location for CA004 |

## Privileged Identity Management: User Administrator

| Setting | Value |
| --- | --- |
| Activation maximum duration | 2 hours |
| On activation require | Azure MFA |
| Justification on activation | Required |
| Approval to activate | Required, approver DKN Lab Admin |
| Permanent eligible assignment | Allowed |
| Permanent active assignment | Not allowed (changed from the default) |
| Expire active assignments after | 15 days (changed from 6 months) |
| MFA on active assignment | Required (changed from the default) |
| Eligible member | Maugaloa Malosi, permanently eligible, scope Directory |
