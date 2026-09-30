# Microsoft Entra ID Conditional Access and Zero Trust Lab

**Four policies. One emergency key. Zero lockouts.**

This is Week 7 of my 12 week IAM Engineer portfolio. I built a Conditional Access framework in Microsoft Entra ID: require MFA for everyone, require a compliant device, block high risk sign ins, and block sign ins from outside the United States with a temporary travel exception. I protected all of it with a break glass account, moved an admin role to just in time access with Privileged Identity Management, proved every policy with five What If simulations, and then cut over from Security Defaults and watched the policies enforce on real sign ins.

Before I could build a single policy, I hit sixteen snags across tenants, account types, licensing, and billing. They are the longest part of this story and the part that taught me the most.

Every claim in this write up is backed by a screenshot or an evidence file in this repo.

---

## The 30 second version

| Question | Answer |
| --- | --- |
| **What problem does this solve?** | A password alone is not enough to prove who someone is. Companies need rules that look at who is signing in, from where, on what device, and how risky the sign in looks, then decide to allow, challenge, or block. Admins also should not hold powerful roles all day long. |
| **What did I build?** | Four Conditional Access policies (three enforced, one in Report only on purpose), a break glass account excluded through one documented group, a named location with a travel exception group, group based licensing, and a Privileged Identity Management setup that makes User Administrator eligible, time limited, MFA protected, and approval based. |
| **How did I prove it works?** | Five What If simulations that changed one variable at a time (5 for 5 passed), a live cutover where the first account challenged for MFA was my own admin account, a regular user forced through MFA registration, a full PIM request, approval, use, and deactivation with its audit trail, and sign in logs showing CA001 as Success for a user and Not Applied for break glass. |
| **Tools** | Microsoft Entra admin center, Microsoft 365 admin center, Microsoft Entra ID P2 (trial), Conditional Access, Named locations, What If tool, Privileged Identity Management, Microsoft Authenticator, Entra sign in and audit logs |
| **Built on** | September 25, 2026 |

---

## Explain it like I am in 5th grade

Picture a school. For years there was one hall monitor who followed a simple rulebook written by the district: everybody shows their ID and a code from their phone. That monitor is **Security Defaults**. It works, but it treats every kid and every door exactly the same.

This lab replaces that monitor with a team of smarter ones. Each one watches for something different: are you really you, are you carrying a school laptop, does something look suspicious, and are you coming in from somewhere the school allows. Before hiring them, I hid a spare key in a fire alarm box so nobody could ever get locked out of the building, even if the new monitors made a mistake.

I also changed how the principal's office works. Instead of a teacher carrying the office key all day, the key stays locked up. When the teacher needs it, she asks, explains why, proves who she is, the principal says yes, and the key disappears on its own after two hours.

| Piece | School version | Real version |
| --- | --- | --- |
| **Security Defaults** | One hall monitor with one rulebook for everyone | Microsoft's free, all or nothing baseline that forces MFA |
| **Conditional Access policy** | A hall monitor who checks one specific thing | An if this, then that rule evaluated at every sign in |
| **Report only** | A monitor in training who writes down what she would have done | A policy that logs its decision without enforcing it |
| **Break glass account** | The spare key in the fire alarm box | An emergency Global Administrator excluded from every policy |
| **Named location** | A map of the places students are allowed to come from | A list of countries or IP ranges Conditional Access can use |
| **Exclusion group** | A permission slip | A group whose members skip one specific policy |
| **Sign in risk** | A helper who watches every school in the world and whispers "this looks suspicious" | Microsoft Entra ID Protection's score for how likely a sign in is not the real person |
| **PIM** | The office key that stays locked up until someone asks for it | Just in time role activation with approval, MFA, and a timer |
| **What If** | A fire drill | A simulation that shows which policies would apply to a sign in |

### Words you will see

| Word | Simple meaning |
| --- | --- |
| Tenant | Your company's own private copy of Microsoft Entra ID |
| Workforce tenant | A tenant built for employees |
| External ID (CIAM) tenant | A tenant built for customers signing into a company's apps |
| UPN | User principal name. The sign in name, like jfawkes@company.onmicrosoft.com |
| Grant control | What a policy requires before it lets someone in, like MFA |
| Eligible assignment | Permission to ask for a role, not the role itself |
| Active assignment | Actually holding the role right now |
| Usage location | The country where a user uses Microsoft services. Required before any license can be assigned |
| Token | The pass a browser keeps after signing in so it does not have to sign in again every minute |

---

## The big picture

```
                    DKN IAM Workforce Lab (Entra ID P2 via LIC_Entra_ID_P2_Users)
                                          │
        ┌─────────────────────────────────┼──────────────────────────────────┐
        │                                 │                                  │
  Conditional Access                 Safety net                        Privileged access
        │                                 │                                  │
  CA001  Require MFA ........ On     breakglass01 (Global Admin)       User Administrator
  CA002  Compliant device ... Report only   │                          Eligible: Maugaloa Malosi
  CA003  High risk = Block .. On     CA_Exclude_BreakGlass              2 hours, MFA,
  CA004  Outside US = Block . On       (excluded from ALL policies)     justification, approval
          │                                 │                          Approver: DKN Lab Admin
          ├── NL_Allowed_Countries (US)     └── CA005 alert: built in
          └── CA_Exclude_Travel_Approved        Week 8 (PowerShell + Graph,
                (Lena Oxton)                    Windows Event Log)
```

---
## Part 1: Getting into the right tenant with the right account

### Situation

I planned to build this in DKN IAM Lab, the tenant I created in Week 3. Conditional Access needs at least Entra ID P1, and risk policies and PIM need P2, so step one was a license. Getting that license turned into a chain of problems, each one a layer deeper than the last.

### Build and snags

**Snag 1: I was in the wrong tenant.** My first screenshot showed Default Directory, one user, and a Free license. The portal had opened the home tenant that Microsoft created automatically for my personal account.

![Default Directory overview](screenshots/001_wrong_tenant_default_directory.png)
*Screenshot 001, Snag: Default Directory, 1 user, 0 groups, Microsoft Entra ID Free. Not the lab tenant.*


**Fix:** switched directories from the Settings menu.

![DKN IAM Lab overview](screenshots/002_switched_to_dkn_iam_lab_tenant.png)
*Screenshot 002, after the fix: DKN IAM Lab, 6 users, 3 groups. Still Microsoft Entra ID Free.*


**Snag 2: the license button was disabled.** In Licenses, All products, **Try / Buy** was grayed out and a banner said license changes now only happen in the Microsoft 365 admin center. No SKUs existed in the tenant.

![Try Buy disabled with M365 redirect banner](screenshots/003_licenses_try_buy_disabled_m365_redirect.png)
*Screenshot 003, Snag: Try / Buy disabled, redirect banner to the M365 admin center, no account SKUs found.*


**Snag 3: my account type was rejected.** The M365 admin center refused me: login is not supported for consumer users without business presence. My Global Admin was a personal Microsoft account visiting the tenant as an external identity. The Entra portal tolerates that. The M365 admin center does not.

![M365 admin center consumer account blocked](screenshots/004_m365_admin_center_consumer_account_blocked.png)
*Screenshot 004, Snag: the M365 admin center rejects a personal Microsoft account.*


**Fix:** created a native cloud admin account, `labadmin`, on the tenant's own onmicrosoft.com domain.

![Create new user form for labadmin](screenshots/005_native_labadmin_create_user_form.png)
*Screenshot 005, Middle: labadmin on the tenant's own domain.*

![Users list with labadmin](screenshots/006_native_labadmin_account_created.png)
*Screenshot 006, Middle: 7 users. Goku Cooper's UPN starts with dknorrington_zoh, the external account fingerprint.*


**Snag 4: the new admin was not an admin.** I did not capture the auto generated password, so I reset it. That reset screen showed **Assigned roles: 0**. The Global Administrator role had never been applied, which is why the M365 admin center said "Unable to process your request."

![labadmin password reset showing zero roles](screenshots/007_labadmin_temporary_password_reset.png)
*Screenshot 007, Snag: a password reset (password blurred) that also exposed Assigned roles: 0.*

![Unable to process your request](screenshots/008_m365_admin_center_unable_to_process_request.png)
*Screenshot 008, Snag: the M365 admin center error for an account with no admin role.*


**Fix:** assigned Global Administrator, waited for it to replicate, and signed in fresh. Security Defaults then forced MFA registration, which became my "before" picture of the old hall monitor.

![Global Administrator assigned to labadmin](screenshots/009_labadmin_global_admin_role_assigned.png)
*Screenshot 009, after the fix: Global Administrator, Direct, Built in.*

![Security defaults MFA prompt](screenshots/010_security_defaults_mfa_registration_prompt.png)
*Screenshot 010, Beginning: Security Defaults requiring MFA registration. This is the control I later replace.*

![Authenticator added for labadmin](screenshots/011_labadmin_authenticator_mfa_registered.png)
*Screenshot 011, Middle: MFA registered for labadmin.*


**Snag 5: the tenant itself was the wrong type.** Even with a native account, a Global Admin role, and MFA, the M365 admin center still refused. The URL ended in `/CIAMTenant`. DKN IAM Lab was an **External ID (customer identity) tenant**, not a workforce tenant. Looking back, the clues were there: its overview showed User flows, Company branding, and Identity providers, which are customer identity features.

![CIAMTenant error page](screenshots/012_m365_admin_center_ciam_tenant_blocked.png)
*Screenshot 012, Snag: External ID users are not supported in the Microsoft 365 admin center.*


When the account was right and it still failed, the problem was one layer up. External tenants also do not support Intune compliance or PIM, so this was not a detour I could work around.

**Fix:** moved the lab to the Default Directory from Snag 1, which turned out to be a workforce tenant, renamed it **DKN IAM Workforce Lab**, and created a native `labadmin` there with the role assigned on the first try.

![Workforce tenant feature highlights](screenshots/013_workforce_tenant_feature_highlights.png)
*Screenshot 013, Middle: the workforce tenant shows Identity Protection, Access reviews, ID Governance, and Global Secure Access.*

![Tenant renamed](screenshots/014_default_directory_renamed_workforce_lab.png)
*Screenshot 014, Middle: renamed to DKN IAM Workforce Lab.*

![Workforce labadmin create form](screenshots/015_workforce_labadmin_create_user_form.png)
*Screenshot 015, Middle: labadmin on dknorringtonzohomail.onmicrosoft.com.*

![Workforce labadmin Global Administrator](screenshots/016_workforce_labadmin_global_admin_assigned.png)
*Screenshot 016, Middle: Global Administrator assigned, with the success toast.*

![Signed into M365 admin center](screenshots/017_m365_admin_center_signed_in_workforce_lab.png)
*Screenshot 017, End of this part: "Good evening, DKN Lab Admin" in DKN IAM Workforce Lab. The door finally opened.*


### Engineer notes

- **Check context before changing anything.** Tenant, account type, and tenant type were three different layers, and each one looked like "access denied."
- **When the account is right and it still fails, go one layer up.** I eliminated account type, role, and MFA one at a time until only the tenant type was left.
- **Admins should use organization owned accounts.** A personal Microsoft account running a tenant is a finding in any real audit.

### Real world mapping

"I can get into Entra but not the M365 admin center" is a common ticket. The answer is usually one of these layers: wrong tenant, external or personal account, missing role, or a tenant type that does not support the feature.

---
## Part 2: Licensing with Entra ID P2

### Situation

Conditional Access needs P1. The risk policy and PIM need P2. I started a free P2 trial and assigned it without leaving anything that could quietly bill me later.

### Build and snags

**Snag 6: the menu moved.** There was no Purchase services under Billing. Trials now live in **Marketplace**.

![Billing menu without Purchase services](screenshots/018_m365_billing_menu_trials_moved_to_marketplace.png)
*Screenshot 018, Snag: the Billing menu. Trials moved to Marketplace.*

![Entra ID P2 product page](screenshots/019_marketplace_entra_id_p2_free_trial_offer.png)
*Screenshot 019, Middle: the trial includes 100 licenses for 1 month.*

![Checkout needs payment verification](screenshots/020_checkout_payment_verification_required.png)
*Screenshot 020, Snag: USD 0.00, but Try now stays disabled until a payment method verifies identity.*


**Snag 7: the checkout still would not submit.** After adding a card, Try now stayed gray. The Sold to address only listed the country. The button unlocked the moment a full organization address was added.

![Checkout with Try now enabled](screenshots/021_checkout_sold_to_address_fixed_try_now_enabled.png)
*Screenshot 021, after the fix: full address added (blurred), Try now enabled.*


**Snag 8: my admin could not manage billing.** The product page said my read permissions limited what I could change. Adding the card created a billing account, and Global Administrator does not automatically own billing. Identity administration and billing administration are separate duties.

![Billing account owner assigned](screenshots/022_labadmin_billing_account_owner_assigned.png)
*Screenshot 022, after the fix: "You're an owner of this billing account."*

![P2 trial active with recurring billing off](screenshots/023_p2_trial_active_recurring_billing_off.png)
*Screenshot 023, End: Active, 0 of 100 assigned, Free trial, **Recurring billing: Off, expires 10/26/2026**. Address blurred.*


**Snag 9: the license would not assign.** "License assignment cannot be done for user with invalid usage location." Microsoft needs a country on the user before any license can be assigned.

![License assignment failed for usage location](screenshots/024_p2_license_assignment_failed_usage_location.png)
*Screenshot 024, Snag: invalid usage location.*

![Usage location United States](screenshots/025_labadmin_usage_location_set_us.png)
*Screenshot 025, after the fix: Usage location set to United States.*

![Assign licenses panel](screenshots/026_p2_assign_licenses_included_services.png)
*Screenshot 026, Middle: P2 includes Entra ID P1, Entra ID P2, and Azure MFA, so one license covers every policy in this lab.*

![1 of 100 assigned](screenshots/027_p2_license_assigned_1_of_100.png)
*Screenshot 027, Middle: 1/100 assigned to DKN Lab Admin.*

![Entra overview P2](screenshots/028_entra_overview_license_p2_confirmed.png)
*Screenshot 028, End: the same overview page as Screenshot 001, now in the right tenant with License: Microsoft Entra ID P2.*


### Engineer notes

- **Recurring billing Off was the first thing I checked.** Some trials that need a payment method convert to paid. I confirmed this one would simply expire.
- **Usage location is a prerequisite, not a detail.** I built it into every account I created after this.

### Real world mapping

License tier questions come up in every Conditional Access project: P1 for Conditional Access, P2 for risk based policies and PIM. So does managing trial cost exposure and keeping billing roles separate from identity roles.

---
## Part 3: Rebuilding the identity population

### Situation

The workforce tenant only had my admin accounts. Policies need people to protect.

### Build

1. Downloaded Microsoft's own bulk create template so the headers would match exactly, then filled in five users with job titles, departments, and **Usage location US** on every row. The departments were chosen for later scenarios: Maugaloa in IT for PIM, Lena in Sales as the traveler, Jamison in Security as a test user. A sanitized copy with passwords removed is in [`evidence/01_bulk_create_users_sanitized.csv`](evidence/01_bulk_create_users_sanitized.csv). The copy with real temporary passwords was deleted after upload.

![Bulk create upload](screenshots/029_bulk_create_users_upload.png)
*Screenshot 029, Middle: 2 users in the list while the file uploads.*

![Bulk operation succeeded](screenshots/030_bulk_create_users_succeeded.png)
*Screenshot 030, Middle: Status Succeeded, Type Create Users.*

![Seven users](screenshots/031_workforce_tenant_users_populated.png)
*Screenshot 031, End: 7 users. Every row worked on the first try because usage location was built into the file.*


2. Created `LIC_Entra_ID_P2_Users` and licensed the **group** instead of each person. Anyone added gets P2 and anyone removed loses it.

![Licensing group members](screenshots/032_lic_entra_id_p2_group_members.png)
*Screenshot 032, Middle: six members. The external Goku Cooper account is left out.*

![Licensing group created](screenshots/033_lic_entra_id_p2_group_created.png)
*Screenshot 033, Middle: Security, Assigned, Cloud.*

![Assign license to group](screenshots/034_group_based_licensing_assign_panel.png)
*Screenshot 034, Middle: assigning P2 to a group of 6.*

![6 of 100 assigned](screenshots/035_group_based_licensing_6_of_100.png)
*Screenshot 035, Middle: 6/100. labadmin had two paths (direct and group) but still counts once.*


3. Removed labadmin's direct license so the group is the single source of truth.

![Only the group assignment remains](screenshots/036_direct_license_removed_group_is_source.png)
*Screenshot 036, End: only the group row remains and the count is still 6.*


### Real world mapping

Group based licensing is how companies avoid orphaned licenses and missed joiners. A mover or leaver changes a group membership, and the license follows automatically.

---

## Part 4: The safety net

### Situation

A Conditional Access mistake can lock every admin out of the tenant, including the person who made it. Before building any policy, I built the way back in.

### Build

1. Created `breakglass01` on the onmicrosoft.com domain (so it still works if a custom domain breaks), with Global Administrator and a long password stored offline. I did not add it to the licensing group. It is an emergency key, not a daily user.

![breakglass01 create form](screenshots/037_breakglass01_create_user_form.png)
*Screenshot 037, Beginning: onmicrosoft.com UPN and a clear display name.*

![breakglass01 usage location](screenshots/038_breakglass01_usage_location_set.png)
*Screenshot 038, Middle: usage location set up front, applying the lesson from Snag 9.*

![breakglass01 Global Administrator](screenshots/039_breakglass01_global_admin_assigned.png)
*Screenshot 039, Middle: Global Administrator, Direct, Built in.*


2. Created `CA_Exclude_BreakGlass`. Every policy excludes this **group**, never an individual account, so the exception lives in one documented place.

![Exclusion group members](screenshots/040_ca_exclude_breakglass_group_members.png)
*Screenshot 040, Middle: only breakglass01 selected.*

![Groups list](screenshots/041_ca_groups_created.png)
*Screenshot 041, Middle: CA_Exclude_BreakGlass and LIC_Entra_ID_P2_Users.*

![breakglass01 MFA registered](screenshots/042_breakglass01_mfa_registered.png)
*Screenshot 042, End: Authenticator registered for breakglass01.*


### Engineer notes

In production, a break glass account usually uses a **FIDO2 security key** kept in a safe, so it does not depend on one person's phone. Authenticator is fine for a lab, and I call out the difference on purpose.

---

## Part 5: Building the policies in Report only

### Situation

If I turned off Security Defaults first, the tenant would have no MFA protection until the policies were finished. I wanted to build and test while Security Defaults kept running, then cut over back to back.

![Security defaults enabled](screenshots/043_security_defaults_enabled_before.png)
*Screenshot 043, Beginning: Security Defaults Enabled, before anything changed.*


**Snag 10, answered by the portal:** every new policy showed "You must first disable security defaults before **enabling** a Conditional Access policy." The Create button still worked in **Report only**. So policies can be built and validated while Security Defaults is on. They just cannot be switched **On** until it is retired.

### CA001: Require MFA for all users

![CA001 include all users](screenshots/044_ca001_users_include_all_users.png)
*Screenshot 044, Middle: Include All users.*

![CA001 exclude break glass](screenshots/045_ca001_users_exclude_breakglass_group.png)
*Screenshot 045, Middle: Exclude CA_Exclude_BreakGlass.*

![CA001 grant and lockout warning](screenshots/046_ca001_grant_mfa_lockout_warning.png)
*Screenshot 046, Middle: Require multifactor authentication, plus the portal's lockout warning.*


**Decision:** the portal offered to exclude my signed in account automatically. I declined and chose "I understand that my account will be impacted." The policy was in Report only, break glass already existed, and an auto exclusion would have created a hidden one off exception outside the group.

![Declined auto exclusion](screenshots/047_ca001_declined_auto_exclusion.png)
*Screenshot 047, Middle: auto exclusion declined.*

![CA001 created](screenshots/048_ca001_created_report_only.png)
*Screenshot 048, End: CA001 in Report only.*


### CA002: Require a compliant device

A compliant device is one that Microsoft Intune has checked and passed. This tenant has no Intune, so if CA002 were turned On it would block everyone. It stays in **Report only** for the whole lab to collect "would have blocked" data.

The portal warned that Report only compliance policies can prompt Mac, iOS, Android, and Linux users for a device certificate and offered to exclude those platforms. I kept all platforms in scope. Excluding them would have quietly turned the policy into "Windows only," and anyone on a phone would walk past it if it were ever switched On.

![CA002 grant](screenshots/049_ca002_grant_require_compliant_device.png)
*Screenshot 049, Middle: Require device to be marked as compliant.*

![CA002 platform and lockout decisions](screenshots/050_ca002_platform_and_lockout_decisions.png)
*Screenshot 050, Middle: Proceed with selected configuration, and I understand.*

![CA002 created](screenshots/051_ca002_created_report_only.png)
*Screenshot 051, End: CA001 and CA002 in Report only.*


### CA003: Block high risk sign ins

This is the first policy that needs P2. Sign in risk is Microsoft's score for whether this particular sign in is really the person, based on signals like impossible travel, anonymous IP addresses, and known attack patterns. I used **Block** because the curriculum calls for the strictest stance. Many companies use MFA plus a password change for high risk instead, so real users can clean up their own accounts.

![Sign in risk High](screenshots/052_ca003_condition_sign_in_risk_high.png)
*Screenshot 052, Middle: Sign in risk configured, High only. User risk is a separate condition.*

![Block access](screenshots/053_ca003_grant_block_access.png)
*Screenshot 053, Middle: Grant set to Block access.*

![Three policies](screenshots/054_ca003_created_report_only.png)
*Screenshot 054, End: three policies in Report only.*


### CA004: Block sign ins from outside allowed countries

Three pieces: a **map** of allowed places, a **permission slip** for approved travel, and the **rule**. The rule reads backwards at first: include every location, exclude the safe one, and block what is left.

![Named location US](screenshots/055_named_location_allowed_countries_us.png)
*Screenshot 055, Beginning: NL_Allowed_Countries, IP based, United States. Unknown countries are left out, which treats unmappable IPs as not allowed.*

![Named locations list](screenshots/056_named_locations_list.png)
*Screenshot 056, Middle: "Not configured in any policy yet."*

![Travel group members](screenshots/057_ca_exclude_travel_approved_members.png)
*Screenshot 057, Middle: CA_Exclude_Travel_Approved with Lena Oxton, described as temporary.*

![Three groups](screenshots/058_ca_groups_three_total.png)
*Screenshot 058, Middle: all three groups.*

![Network include any](screenshots/059_ca004_network_include_any_location.png)
*Screenshot 059, Middle: Network, Include any network or location. The banner says Locations is moving to Network, so this reflects the current portal.*

![Network exclude allowed](screenshots/060_ca004_network_exclude_allowed_countries.png)
*Screenshot 060, Middle: Exclude NL_Allowed_Countries.*

![Two exclusion groups](screenshots/061_ca004_users_two_exclusion_groups.png)
*Screenshot 061, Middle: users excluded through two groups.*

![Four policies](screenshots/062_ca004_created_four_policies.png)
*Screenshot 062, Middle: four policies in Report only.*

![CA004 verified](screenshots/063_ca004_verified_block_access.png)
*Screenshot 063, End: I reopened CA004 to confirm Block access. The one condition is the same location shown in both places during Microsoft's move.*

![Named location linked](screenshots/064_named_location_linked_to_ca004.png)
*Screenshot 064, End: the named location now shows CA004. Compare with Screenshot 056.*


### Engineer notes

- **Block always wins.** When several policies apply and one says Block, the user is blocked no matter what the others allow.
- **Numbered names (CA001, CA002) make policies easy to reference** in change tickets, audits, and troubleshooting.
- **Exceptions loosen one rule, not all of them.** Lena skips CA004 but still needs MFA.

---

## Part 6: CA005, the break glass alarm

### Situation

The break glass account skips every policy, which makes it the most powerful and least watched account in the tenant. The curriculum calls for an alert when it is used. The standard build is Entra sign in logs sent to a Log Analytics workspace with an alert rule, which needs an Azure subscription.

![No Azure subscriptions](screenshots/065_azure_subscriptions_workforce_tenant_empty.png)
*Screenshot 065, Snag: 0 subscriptions in the workforce tenant. The Week 3 subscription belongs to the old External ID tenant.*


### Decision

I compared three options: a new Pay as you go subscription (small cost, real enterprise pattern), moving the Week 3 subscription across tenants (wipes its role assignments, generally avoided), or building the alert with PowerShell and Microsoft Graph. I chose **PowerShell and Graph in Week 8** to keep the lab at $0. Until then, an interim control is in place: a saved sign in log filter.

**Snag 11: the interim control found nothing.** breakglass01 had definitely signed in when it registered MFA, yet a filter for `breakglass01` returned no sign ins. A detection that quietly finds nothing is worse than none.

![Filter breakglass01 no results](screenshots/066_ca005_interim_filter_no_results.png)
*Screenshot 066, Snag: "No sign ins found."*

![Filter Break Glass found](screenshots/067_ca005_interim_filter_breakglass_found.png)
*Screenshot 067, after the fix: filtering on "Break Glass" found both sign ins. IP and city blurred.*


**Root cause:** this view's User filter matches the display name (Break Glass Emergency Access 01), not the username. The Week 8 script will query on userPrincipalName, because a display name can be changed by any admin and would let someone dodge the alert by renaming the account. The two entries also told a clean story: error 50055 was the expected "temporary password must be changed" interruption, followed by a success a minute later.

**Known constraint for Week 8:** this tenant has no Exchange mailbox, so the alert cannot email through Graph. It will write to a log, the Windows Event Log, or a webhook instead.

### Update: CA005 is built

I built the alert in Week 8 and kept both promises from this section. The script, `Watch-BreakGlassSignIn.ps1`, queries the Entra sign in logs through Microsoft Graph by **userPrincipalName**, not display name, and writes the alert to the **Windows Event Log** (Event ID 9001 when the account is used, 9002 when there are only failed attempts), plus an evidence CSV. It uses read only Graph scopes consented for my admin account only.

In testing it caught a real break glass sign in: 5 log entries in 24 hours, which triage showed were 2 actual sign ins and 0 attacks. Every entry showed Conditional Access **notApplied**, which is this lab's exclusion group working exactly as designed. The full build, screenshots, and triage are in [powershell_iam_automation](https://github.com/GokuCooper/powershell_iam_automation) (Part 6, Screenshots 40 to 45).

---

## Part 7: Just in time admin access with PIM

### Situation

In Week 3, Maugaloa Malosi had User Administrator active all day, every day. If her account were stolen, so was the role. In Week 7 the same person gets the same role only when she asks, explains why, passes MFA, and gets approval, for two hours.

### Build

![PIM activation settings](screenshots/068_pim_user_admin_activation_settings.png)
*Screenshot 068, Middle: 2 hours maximum, Azure MFA, justification required, approval required, approver DKN Lab Admin.*


**Snag 12: a loophole in the defaults.** PIM allowed **permanent active assignment** by default. Any admin could skip the activation workflow and hand someone User Administrator forever with no approval or MFA.

![Assignment settings default](screenshots/069_pim_user_admin_assignment_settings_default.png)
*Screenshot 069, Snag: Allow permanent active assignment checked, MFA on active assignment unchecked.*

![Assignment settings hardened](screenshots/070_pim_user_admin_assignment_settings_hardened.png)
*Screenshot 070, after the fix: permanent active off, active assignments expire after 15 days, MFA required on active assignment.*

![Add assignment membership](screenshots/071_pim_add_assignment_maugaloa_membership.png)
*Screenshot 071, Middle: User Administrator, scope Directory, member Maugaloa. An Administrative unit scope would narrow her further in a larger company.*

![Eligible permanent](screenshots/072_pim_assignment_setting_eligible_permanent.png)
*Screenshot 072, Middle: Eligible, Permanently eligible.*

![Maugaloa eligible](screenshots/073_pim_maugaloa_eligible_confirmed.png)
*Screenshot 073, End: Maugaloa on the Eligible assignments tab.*


### The end to end test

I played two people in two different browsers so sessions could not collide: Maugaloa asking, and labadmin approving.

![Maugaloa zero roles](screenshots/074_maugaloa_password_reset_zero_roles.png)
*Screenshot 074, Beginning: Maugaloa's profile shows Assigned roles: 0 while she is eligible. Eligible is not access. Password blurred.*

![Activation request](screenshots/075_pim_maugaloa_activation_request.png)
*Screenshot 075, Middle: 2 hours, reason "Reset password for Sales user Lena Oxton, help desk ticket LAB-001."*

![Pending approval](screenshots/076_pim_activation_pending_approval.png)
*Screenshot 076, Middle: Pending approval.*

![labadmin approves](screenshots/077_pim_labadmin_approves_request.png)
*Screenshot 077, Middle: a different person reads the reason and approves with "Approved for LAB-001."*

![Role active](screenshots/078_pim_maugaloa_role_active_2_hours.png)
*Screenshot 078, Middle: Activated, ending 12:30:55 AM. The two hour window restarted at approval, not at the request.*

![Maugaloa resets Lena's password](screenshots/079_pim_maugaloa_resets_lena_password.png)
*Screenshot 079, Middle: she completes exactly the task in her ticket. Password blurred.*


**Snag 13: the role could not be handed back right away.** "The Active duration is too short. Minimum required is 5 minutes." PIM makes a role stay active at least five minutes so the change can finish spreading and the audit record stays consistent.

![Deactivate request](screenshots/080_pim_maugaloa_deactivate_request.png)
*Screenshot 080, Middle: the deactivate panel shows the start time 10:30:56 PM.*

![Deactivate failed](screenshots/081_pim_deactivate_failed_5_minute_minimum.png)
*Screenshot 081, Snag: the 5 minute minimum.*

![Role deactivated](screenshots/082_pim_maugaloa_role_deactivated.png)
*Screenshot 082, after the fix: Active assignments shows No results. Still eligible, zero power.*

![PIM audit trail](screenshots/083_pim_audit_trail.png)
*Screenshot 083, End: the full chain in Resource audit.*


Read the audit from the bottom up and it is the whole story:

| Time | Who | What happened |
| --- | --- | --- |
| 10:17:26 PM | Goku Cooper | Updated the User Administrator role settings |
| 10:19:20 PM | Goku Cooper | Made Maugaloa permanently eligible |
| 10:27:00 PM | Maugaloa | Requested activation |
| 10:30:55 PM | DKN Lab Admin | Approved the request |
| 10:31:02 PM | Maugaloa | Role activated |
| 10:33:10 to 10:35:28 PM | Maugaloa | Four failed deactivation attempts, all under 5 minutes |
| 10:37:15 PM | Maugaloa | Deactivation completed |

**An honest note:** the first rows say Goku Cooper because I configured PIM while signed in as the tenant's original personal account. In production, every admin change would come from an organization owned admin account so the audit trail maps to an accountable identity.

### Real world mapping

This audit table is what an auditor asks for in a SOX access review: who had privileged access, why, who approved it, and for how long. It connects directly to Week 11.

---

## Part 8: Proving it with What If

### Situation

Before turning anything On, I wanted proof that each policy does exactly what it should and that break glass can never be caught.

**Snag 14:** the What If tool would not accept a country by itself. It requires an IP address that maps to that country, because CA004 looks up the country from the IP. I used public IPs that are safe to show: `8.8.8.8` (Google, United States) and `200.160.2.3` (NIC.br, Brazil).

![Country requires IP](screenshots/084_whatif_country_requires_matching_ip.png)
*Screenshot 084, Snag: "If using an IP address or Country, both fields will be required and should correctly map together."*


Every test used the same app, platform, and client, and changed one variable at a time. What If does evaluate Report only policies and labels their state.

| Test | User | Conditions | Expected | Actual | Pass |
| --- | --- | --- | --- | --- | --- |
| 1 | Jamison | US, no risk | CA001 and CA002 apply | CA001 and CA002 apply. CA003 skipped (sign in risk). CA004 skipped (location) | ✅ |
| 2 | Jamison | Brazil, no risk | CA004 blocks | CA001, CA002, CA004 apply. Block wins | ✅ |
| 3 | Lena | Brazil, no risk | CA004 skipped (travel exception) | CA004 skipped, reason users and groups. CA001 and CA002 still apply | ✅ |
| 4 | Vaira | US, High risk | CA003 blocks | CA001, CA002, CA003 apply. Block wins. CA004 skipped (location) | ✅ |
| 5 | breakglass01 | Brazil, High risk | Nothing applies | 0 policies apply. All four skipped, reason users and groups | ✅ |

Full results are also in [`evidence/03_whatif_and_signin_log_results.md`](evidence/03_whatif_and_signin_log_results.md).

### Test 1: normal employee, normal day

![What If test 1](screenshots/085_whatif_test1_jamison_us_inputs.png)
*Screenshot 085, Beginning: Jamison, United States, 8.8.8.8, no risk.*

![What If test 1](screenshots/086_whatif_test1_policies_apply.png)
*Screenshot 086, Middle: CA001 and CA002 apply, both Report only.*

![What If test 1](screenshots/087_whatif_test1_policies_not_apply.png)
*Screenshot 087, End: CA003 skipped for sign in risk, CA004 skipped for location. The reason column says why, not just no.*


### Test 2: the same employee from Brazil

![What If test 2](screenshots/088_whatif_test2_jamison_brazil_inputs.png)
*Screenshot 088, Beginning: only the IP and country changed.*

![What If test 2](screenshots/089_whatif_test2_ca004_blocks.png)
*Screenshot 089, Middle: CA004 moves to "will apply" with Block access.*

![What If test 2](screenshots/090_whatif_test2_policies_not_apply.png)
*Screenshot 090, End: only CA003 is skipped.*


### Test 3: an approved traveler from the same place

![What If test 3](screenshots/091_whatif_test3_lena_brazil_inputs.png)
*Screenshot 091, Beginning: Lena, same Brazil IP. Only the user changed from Test 2.*

![What If test 3](screenshots/092_whatif_test3_lena_policies_apply.png)
*Screenshot 092, Middle: CA001 and CA002 still apply. She still needs MFA abroad.*

![What If test 3](screenshots/093_whatif_test3_lena_ca004_excluded.png)
*Screenshot 093, End: CA004 skipped with the reason "Users and groups." Same place, different person, different result.*


### Test 4: a high risk sign in from a normal place

![What If test 4](screenshots/094_whatif_test4_vaira_high_risk_inputs.png)
*Screenshot 094, Beginning: Vaira, United States, Sign in risk High.*

![What If test 4](screenshots/095_whatif_test4_vaira_ca003_blocks.png)
*Screenshot 095, Middle: CA003 applies with Block access.*

![What If test 4](screenshots/096_whatif_test4_vaira_policies_not_apply.png)
*Screenshot 096, End: CA004 skipped for location. One variable changed and exactly one policy reacted.*


### Test 5: break glass on the worst possible day

![What If test 5](screenshots/097_whatif_test5_breakglass_worst_case_inputs.png)
*Screenshot 097, Beginning: breakglass01, Brazil, High risk.*

![What If test 5](screenshots/098_whatif_test5_breakglass_no_policies_apply.png)
*Screenshot 098, Middle: 0 policies found. No policies.*

![What If test 5](screenshots/099_whatif_test5_breakglass_all_excluded.png)
*Screenshot 099, End: all four policies skipped, each for "Users and groups." The emergency key works even when everything else looks wrong.*


---

## Part 9: The cutover and live enforcement

### Situation

Security Defaults off, then CA001, CA003, and CA004 On, back to back. CA002 stays in Report only.

![Security defaults disable panel](screenshots/100_cutover_security_defaults_disable_panel.png)
*Screenshot 100, Beginning: Disabled, with Microsoft's warning that the organization is vulnerable.*

![Not protected by security defaults](screenshots/101_cutover_security_defaults_disabled.png)
*Screenshot 101, Middle: "Your organization is not protected by security defaults." True for about a minute.*

![CA001 On](screenshots/102_cutover_ca001_switched_on.png)
*Screenshot 102, Middle: CA001 switched On, with "I understand" selected again.*

![CA003 On](screenshots/103_cutover_ca003_switched_on.png)
*Screenshot 103, Middle: CA003 switched On.*

![CA004 On](screenshots/104_cutover_ca004_switched_on.png)
*Screenshot 104, Middle: CA004 switched On.*

![Final policy states](screenshots/105_cutover_policies_on_ca002_report_only.png)
*Screenshot 105, End: CA001, CA003, CA004 On and CA002 Report only, all modified at 10:57 PM.*


### What happened

**The first account challenged was mine.** The zohomail admin account had never registered MFA in this tenant. The moment CA001 went live, it was sent to register before continuing. That proves the decision in Part 5: had I accepted the portal's auto exclusion, my own admin account would have skipped MFA forever.

![Admin forced to register MFA](screenshots/106_live_ca001_admin_mfa_registration.png)
*Screenshot 106, Middle: the external admin account (#EXT#) registering Authenticator after CA001 went live.*


**Snag 15: an already signed in session broke.** The portal showed "Interaction required," error **AADSTS50076**: due to a configuration change made by your administrator, you must use MFA to access `00000003-0000-0000-c000-000000000000`, which is Microsoft Graph.

![AADSTS50076](screenshots/107_live_ca001_session_reauth_aadsts50076.png)
*Screenshot 107, Snag: the portal's background token renewal was refused.*


**Root cause:** my session token was issued before CA001 existed and only carried a password. When the portal tried to renew it silently, Conditional Access now required MFA, and a silent renewal cannot show an MFA prompt. **Fix:** signed in again and completed MFA.

![Portal restored](screenshots/108_live_admin_reauth_portal_restored.png)
*Screenshot 108, after the fix: the portal loads normally.*


**A regular user:** Jamison signed in with his password and CA001 stopped him until he registered MFA.

![Let's keep your account secure](screenshots/109_live_jamison_mfa_registration_required.png)
*Screenshot 109, Middle: jfawkes is told to set up another way to verify.*

![Install Authenticator](screenshots/110_live_jamison_install_authenticator.png)
*Screenshot 110, Middle: registration flow.*

![Authenticator added for Jamison](screenshots/111_live_jamison_authenticator_added.png)
*Screenshot 111, Middle: Authenticator app added.*

![Jamison My Apps](screenshots/112_live_jamison_signed_in_myapps.png)
*Screenshot 112, End: Jamison signed in to My Apps after MFA.*

![Jamison denied Conditional Access](screenshots/113_live_jamison_denied_conditional_access_no_role.png)
*Screenshot 113, End: Jamison opening Conditional Access gets "You don't have access." MFA proved who he is. His lack of a role decides what he can touch. That is authentication versus authorization.*


**Break glass with the policies live:**

![breakglass01 signed in](screenshots/114_live_breakglass_signed_in.png)
*Screenshot 114, End: breakglass01 signed in.*

![breakglass01 admin access](screenshots/115_live_breakglass_admin_access_verified.png)
*Screenshot 115, End: breakglass01 opens Conditional Access with full access.*


---

## Part 10: Hardening and the receipts

### Restricting the admin center

Jamison could open the Entra admin center home page even though he could not change anything. I set **Restrict access to Microsoft Entra admin center** to Yes.

![Admin center visible before](screenshots/116_hardening_jamison_admin_center_visible_before.png)
*Screenshot 116, Beginning: a regular user can browse the admin center home page.*

![Restrict admin center setting](screenshots/117_hardening_restrict_admin_center_non_admins.png)
*Screenshot 117, Middle: Restrict access set to Yes.*

![Jamison blocked from admin center](screenshots/118_hardening_jamison_blocked_from_admin_center.png)
*Screenshot 118, End: Jamison gets "You don't have access."*


**An honest caveat:** this only hides the portal. It does not block PowerShell or Microsoft Graph. What actually stops Jamison from changing policy is that he holds no admin role (Screenshot 113). This setting is a tidiness layer on top of that, not a security boundary.

### The receipts: sign in logs

Screenshots show what users saw. The sign in logs show what Entra decided.

![Jamison sign in log](screenshots/119_signin_log_jamison_ca_success.png)
*Screenshot 119, End: Jamison at 11:09:45 PM, Interrupted 50055 (temporary password) with Conditional Access Not Applied because the sign in never reached evaluation. At 11:14:59 PM, Success with **Conditional Access: Success** and a requirement of Multifactor authentication. IP and city blurred.*

![Break glass sign in log](screenshots/120_signin_log_breakglass_ca_not_applied.png)
*Screenshot 120, End: every breakglass01 sign in shows **Conditional Access: Not Applied**. Two show a requirement of **Single factor authentication**.*


That last detail matters. With Security Defaults off and break glass excluded from CA001, nothing in the tenant forces MFA for the emergency account. That is the intended tradeoff of an emergency key, and it is exactly why the account needs strong compensating controls, listed below.

---

## Final result

| Deliverable | Status | Evidence |
| --- | --- | --- |
| Right tenant and admin account | Working after five layered snags: tenant, license menu, account type, missing role, tenant type | Screenshots 001 to 017 |
| Entra ID P2 | Trial active, recurring billing off, group based licensing, 6 of 100 assigned | Screenshots 018 to 036 |
| Break glass | Global Admin, onmicrosoft.com UPN, MFA registered, excluded from every policy through one group | Screenshots 037 to 042, 097 to 099, 114, 115, 120 |
| CA001 Require MFA | On. Enforced on the admin who enabled it and on a regular user | Screenshots 044 to 048, 102, 106, 109 to 112, 119 |
| CA002 Compliant device | Report only by design (no Intune) | Screenshots 049 to 051, 105 |
| CA003 Block high risk | On. Proven to block in simulation | Screenshots 052 to 054, 094 to 096, 103 |
| CA004 Block outside US | On, with a working travel exception | Screenshots 055 to 064, 088 to 093, 104 |
| CA005 Break glass alert | Built in Week 8 with PowerShell and Microsoft Graph. Detects every break glass sign in, writes Event 9001 or 9002, and saves evidence. See [powershell_iam_automation](https://github.com/GokuCooper/powershell_iam_automation) | Screenshots 065 to 067, Week 8 Screenshots 40 to 45 |
| PIM for User Administrator | Eligible, 2 hours, MFA, justification, approval, loophole closed, full lifecycle audited | Screenshots 068 to 083 |
| Validation | What If 5 for 5, live enforcement, sign in log evidence | Screenshots 084 to 120 |

---

## Troubleshooting summary

| # | Snag | Layer | Root cause | Fix |
| --- | --- | --- | --- | --- |
| 1 | Default Directory, Free license | Tenant context | Portal opened the account's home tenant | Switched directories |
| 2 | Try / Buy disabled | Platform change | Licensing moved to the M365 admin center | Used the M365 admin center |
| 3 | Consumer user rejected | Account type | Personal Microsoft account is external to the tenant | Created a native admin account |
| 4 | Unable to process your request | Authorization | Global Administrator never assigned | Assigned the role |
| 5 | /CIAMTenant error | Tenant type | Lab tenant was External ID, not workforce | Moved to the workforce tenant |
| 6 | No Purchase services | Platform change | Trials moved to Marketplace | Used Marketplace |
| 7 | Try now stayed disabled | Billing data | Sold to address was incomplete | Added a full address |
| 8 | Read only billing page | Separation of duties | Global Admin is not a billing role | Assigned billing account owner |
| 9 | Invalid usage location | User data | No country on the user | Set usage location, then built it into every account |
| 10 | Must disable Security Defaults to enable | Platform rule | Only Report only allowed while defaults are on | Built in Report only, cut over later |
| 11 | Log filter found nothing | Detection design | Filter matches display name, not UPN | Filtered on display name. The Week 8 script queries by UPN |
| 12 | Permanent active allowed | PIM defaults | Default role settings allow bypass | Disabled permanent active, required MFA |
| 13 | Deactivate failed | PIM rule | 5 minute minimum active duration | Waited past 5 minutes |
| 14 | What If rejected country | Tool rule | Country needs a matching IP | Used public IPs that map to each country |
| 15 | AADSTS50076 after cutover | Session token | Token issued before CA001 had single factor only | Signed in again with MFA |
| 16 | Admin forced into MFA registration | Policy scope, working as designed | CA001 applies to everyone, including the admin | Registered MFA. Proved no hidden exclusions |

---

## Verification log

| Area | What could have hidden a problem | How I checked | What the evidence showed |
| --- | --- | --- | --- |
| Tenant | Two tenants with similar names and one wrong type | Overview page, then the M365 admin center URL | /CIAMTenant exposed the wrong type |
| Licensing | A trial can quietly convert to paid | Subscription details page | Recurring billing Off, expires 10/26/2026 |
| Roles | A created admin might not hold a role | The user's Assigned roles count | Caught 0 roles on the first labadmin |
| Policies | Report only never proves enforcement | What If, then live sign ins, then sign in logs | 5 for 5 in simulation, CA001 Success in the log |
| Break glass | An emergency account that gets caught is useless | Worst case What If, then a live sign in | 0 policies applied in both |
| PIM | Settings can look right and still have gaps | Assignment tab, then a full activation and the audit | Loophole found and closed, lifecycle recorded |
| Monitoring | A detection that returns nothing looks fine | Tested against a known event | Filter bug found and root caused |

---

## What I would change before calling this production ready

- **~~Build CA005 for real (Week 8)~~ Done.** Built in [powershell_iam_automation](https://github.com/GokuCooper/powershell_iam_automation), querying sign ins by userPrincipalName and alerting to the Windows Event Log. Next step: run it unattended with an app registration and certificate, and forward the event to a monitored channel.
- **Protect break glass with phishing resistant MFA.** A FIDO2 security key stored in a safe, plus a dedicated policy that requires an authentication strength for the break glass group only, instead of relying on a phone.
- **Add a second break glass account**, so one lost key is not a single point of failure.
- **Move CA001 to an authentication strength** that requires phishing resistant methods for admins.
- **Enroll devices in Intune** so CA002 can be switched On.
- **Block legacy authentication** with its own policy, since older protocols cannot do MFA.
- **Put CA_Exclude_Travel_Approved under an access review** so travel exceptions expire automatically (Week 11).
- **Scope PIM with Administrative units** so a help desk admin only manages their own department.
- **Set "Users can register applications" to No** to reduce consent phishing risk.
- **Make every admin change from organization owned accounts**, never a personal Microsoft account.
- **Announce cutovers to users ahead of time**, because existing sessions get challenged too (Snag 15).

---

## Interview talking points (Problem, Solution, Impact, Learning)

**"Walk me through a Conditional Access rollout."**

- **Problem:** The tenant relied on Security Defaults, which treats everyone the same and cannot express location, device, or risk rules.
- **Solution:** I built four policies in Report only while Security Defaults kept protecting the tenant, protected everything with a break glass account excluded through one group, validated each policy with What If changing one variable at a time, then cut over in a single minute.
- **Impact:** Five for five simulations passed, the first live MFA challenge landed on my own admin account (proving there were no hidden exclusions), and the sign in logs show CA001 as Success for a real user.
- **Learning:** Build and test before you enforce, and treat your own account like everyone else's.

**"Tell me about a time the problem was not where it looked."**

- **Problem:** My admin account could not open the Microsoft 365 admin center.
- **Solution:** I worked up one layer at a time: tenant, account type, role, MFA. When a native Global Admin with MFA was still rejected, the URL ended in /CIAMTenant. The tenant itself was an External ID tenant.
- **Impact:** I moved the lab to a workforce tenant and everything that depended on it, licensing, PIM, and compliance, became possible.
- **Learning:** When the account is right and it still fails, go one layer up.

**"If two Conditional Access policies conflict, which wins?"** Block always wins. In my Test 2, CA001 required MFA and CA004 said Block. The user is blocked no matter how well they pass MFA.

**"What is the difference between user risk and sign in risk?"** User risk is how likely the account is compromised. Sign in risk is how likely this one sign in is not the real person. CA003 uses sign in risk.

**"How do you prevent locking yourself out?"** A cloud only break glass account on the onmicrosoft.com domain, excluded through a single group, tested under worst case conditions before any policy goes On, and monitored whenever it is used.

**"Why use PIM if the person is trusted?"** Trust is not the risk. A stolen session is. With PIM the role does not exist on the account most of the time, and when it does, there is a reason, an approver, MFA, a two hour timer, and an audit trail.

---

## Resume bullet

> Designed and deployed a 4 policy Microsoft Entra ID Conditional Access framework (MFA for all users, compliant device in report only, high risk sign in block, and location block with a travel exception), protected by a group excluded break glass account and validated with 5 What If simulations before cutover from Security Defaults.
>
> Implemented Privileged Identity Management just in time access for User Administrator with 2 hour activation, MFA, justification, and approval, closed a default permanent assignment loophole, and documented the full request to deactivation lifecycle in the audit log. Resolved 16 issues across tenant type, account type, licensing, billing, and session tokens.

*Only add a percentage or time saved figure if you have a measured baseline to compare against.*

---

## Repository layout

```
entra_conditional_access_zero_trust/
├── README.md
├── evidence/
│   ├── 01_bulk_create_users_sanitized.csv
│   ├── 02_conditional_access_policy_matrix.md
│   └── 03_whatif_and_signin_log_results.md
└── screenshots/
    ├── 001_wrong_tenant_default_directory.png
    ├── 002_switched_to_dkn_iam_lab_tenant.png
    ├── 003_licenses_try_buy_disabled_m365_redirect.png
    ├── 004_m365_admin_center_consumer_account_blocked.png
    ├── 005_native_labadmin_create_user_form.png
    ├── 006_native_labadmin_account_created.png
    ├── 007_labadmin_temporary_password_reset.png
    ├── 008_m365_admin_center_unable_to_process_request.png
    ├── 009_labadmin_global_admin_role_assigned.png
    ├── 010_security_defaults_mfa_registration_prompt.png
    ├── 011_labadmin_authenticator_mfa_registered.png
    ├── 012_m365_admin_center_ciam_tenant_blocked.png
    ├── 013_workforce_tenant_feature_highlights.png
    ├── 014_default_directory_renamed_workforce_lab.png
    ├── 015_workforce_labadmin_create_user_form.png
    ├── 016_workforce_labadmin_global_admin_assigned.png
    ├── 017_m365_admin_center_signed_in_workforce_lab.png
    ├── 018_m365_billing_menu_trials_moved_to_marketplace.png
    ├── 019_marketplace_entra_id_p2_free_trial_offer.png
    ├── 020_checkout_payment_verification_required.png
    ├── 021_checkout_sold_to_address_fixed_try_now_enabled.png
    ├── 022_labadmin_billing_account_owner_assigned.png
    ├── 023_p2_trial_active_recurring_billing_off.png
    ├── 024_p2_license_assignment_failed_usage_location.png
    ├── 025_labadmin_usage_location_set_us.png
    ├── 026_p2_assign_licenses_included_services.png
    ├── 027_p2_license_assigned_1_of_100.png
    ├── 028_entra_overview_license_p2_confirmed.png
    ├── 029_bulk_create_users_upload.png
    ├── 030_bulk_create_users_succeeded.png
    ├── 031_workforce_tenant_users_populated.png
    ├── 032_lic_entra_id_p2_group_members.png
    ├── 033_lic_entra_id_p2_group_created.png
    ├── 034_group_based_licensing_assign_panel.png
    ├── 035_group_based_licensing_6_of_100.png
    ├── 036_direct_license_removed_group_is_source.png
    ├── 037_breakglass01_create_user_form.png
    ├── 038_breakglass01_usage_location_set.png
    ├── 039_breakglass01_global_admin_assigned.png
    ├── 040_ca_exclude_breakglass_group_members.png
    ├── 041_ca_groups_created.png
    ├── 042_breakglass01_mfa_registered.png
    ├── 043_security_defaults_enabled_before.png
    ├── 044_ca001_users_include_all_users.png
    ├── 045_ca001_users_exclude_breakglass_group.png
    ├── 046_ca001_grant_mfa_lockout_warning.png
    ├── 047_ca001_declined_auto_exclusion.png
    ├── 048_ca001_created_report_only.png
    ├── 049_ca002_grant_require_compliant_device.png
    ├── 050_ca002_platform_and_lockout_decisions.png
    ├── 051_ca002_created_report_only.png
    ├── 052_ca003_condition_sign_in_risk_high.png
    ├── 053_ca003_grant_block_access.png
    ├── 054_ca003_created_report_only.png
    ├── 055_named_location_allowed_countries_us.png
    ├── 056_named_locations_list.png
    ├── 057_ca_exclude_travel_approved_members.png
    ├── 058_ca_groups_three_total.png
    ├── 059_ca004_network_include_any_location.png
    ├── 060_ca004_network_exclude_allowed_countries.png
    ├── 061_ca004_users_two_exclusion_groups.png
    ├── 062_ca004_created_four_policies.png
    ├── 063_ca004_verified_block_access.png
    ├── 064_named_location_linked_to_ca004.png
    ├── 065_azure_subscriptions_workforce_tenant_empty.png
    ├── 066_ca005_interim_filter_no_results.png
    ├── 067_ca005_interim_filter_breakglass_found.png
    ├── 068_pim_user_admin_activation_settings.png
    ├── 069_pim_user_admin_assignment_settings_default.png
    ├── 070_pim_user_admin_assignment_settings_hardened.png
    ├── 071_pim_add_assignment_maugaloa_membership.png
    ├── 072_pim_assignment_setting_eligible_permanent.png
    ├── 073_pim_maugaloa_eligible_confirmed.png
    ├── 074_maugaloa_password_reset_zero_roles.png
    ├── 075_pim_maugaloa_activation_request.png
    ├── 076_pim_activation_pending_approval.png
    ├── 077_pim_labadmin_approves_request.png
    ├── 078_pim_maugaloa_role_active_2_hours.png
    ├── 079_pim_maugaloa_resets_lena_password.png
    ├── 080_pim_maugaloa_deactivate_request.png
    ├── 081_pim_deactivate_failed_5_minute_minimum.png
    ├── 082_pim_maugaloa_role_deactivated.png
    ├── 083_pim_audit_trail.png
    ├── 084_whatif_country_requires_matching_ip.png
    ├── 085_whatif_test1_jamison_us_inputs.png
    ├── 086_whatif_test1_policies_apply.png
    ├── 087_whatif_test1_policies_not_apply.png
    ├── 088_whatif_test2_jamison_brazil_inputs.png
    ├── 089_whatif_test2_ca004_blocks.png
    ├── 090_whatif_test2_policies_not_apply.png
    ├── 091_whatif_test3_lena_brazil_inputs.png
    ├── 092_whatif_test3_lena_policies_apply.png
    ├── 093_whatif_test3_lena_ca004_excluded.png
    ├── 094_whatif_test4_vaira_high_risk_inputs.png
    ├── 095_whatif_test4_vaira_ca003_blocks.png
    ├── 096_whatif_test4_vaira_policies_not_apply.png
    ├── 097_whatif_test5_breakglass_worst_case_inputs.png
    ├── 098_whatif_test5_breakglass_no_policies_apply.png
    ├── 099_whatif_test5_breakglass_all_excluded.png
    ├── 100_cutover_security_defaults_disable_panel.png
    ├── 101_cutover_security_defaults_disabled.png
    ├── 102_cutover_ca001_switched_on.png
    ├── 103_cutover_ca003_switched_on.png
    ├── 104_cutover_ca004_switched_on.png
    ├── 105_cutover_policies_on_ca002_report_only.png
    ├── 106_live_ca001_admin_mfa_registration.png
    ├── 107_live_ca001_session_reauth_aadsts50076.png
    ├── 108_live_admin_reauth_portal_restored.png
    ├── 109_live_jamison_mfa_registration_required.png
    ├── 110_live_jamison_install_authenticator.png
    ├── 111_live_jamison_authenticator_added.png
    ├── 112_live_jamison_signed_in_myapps.png
    ├── 113_live_jamison_denied_conditional_access_no_role.png
    ├── 114_live_breakglass_signed_in.png
    ├── 115_live_breakglass_admin_access_verified.png
    ├── 116_hardening_jamison_admin_center_visible_before.png
    ├── 117_hardening_restrict_admin_center_non_admins.png
    ├── 118_hardening_jamison_blocked_from_admin_center.png
    ├── 119_signin_log_jamison_ca_success.png
    └── 120_signin_log_breakglass_ca_not_applied.png
```

*Built as part of the IAM Engineer Mentorship Program, Week 7. This is a lab environment with test data only. IP addresses, street addresses, and passwords are blurred.*
