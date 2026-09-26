# Validation results

## What If simulations (all policies in Report only at the time)

App: Office 365 SharePoint Online. Platform: Windows. Client app: Browser. One variable changed per test.

| Test | User | Conditions | Expected | Actual | Pass |
| --- | --- | --- | --- | --- | --- |
| 1 | Jamison Fawkes | United States (8.8.8.8), no risk | CA001 and CA002 apply | CA001 and CA002 apply. CA003 skipped (reason: sign in risk). CA004 skipped (reason: location) | Yes |
| 2 | Jamison Fawkes | Brazil (200.160.2.3), no risk | CA004 blocks | CA001, CA002, and CA004 apply. Block wins. CA003 skipped (sign in risk) | Yes |
| 3 | Lena Oxton | Brazil (200.160.2.3), no risk | CA004 skipped because of the travel exception | CA004 skipped (reason: users and groups). CA001 and CA002 still apply | Yes |
| 4 | Vaira Singhania | United States (8.8.8.8), High sign in risk | CA003 blocks | CA001, CA002, and CA003 apply. Block wins. CA004 skipped (location) | Yes |
| 5 | breakglass01 | Brazil (200.160.2.3), High sign in risk | No policy applies | 0 policies apply. All four skipped (reason: users and groups) | Yes |

## Live sign in log evidence after cutover (September 25, 2026)

IP address and city are blurred in the screenshots. Country (US) is left visible because CA004 depends on it.

### Jamison Fawkes

| Time | Application | Status | Error code | Conditional Access | Authentication requirement |
| --- | --- | --- | --- | --- | --- |
| 11:09:45 PM | Azure Portal | Interrupted | 50055 (temporary password had to be changed) | Not Applied | Multifactor authentication |
| 11:14:59 PM | Azure Portal | Success | 0 | Success | Multifactor authentication |

### Break Glass Emergency Access 01

| Time | Application | Status | Error code | Conditional Access | Authentication requirement |
| --- | --- | --- | --- | --- | --- |
| 11:14:40 PM | Azure Portal | Success | 0 | Not Applied | Single factor authentication |
| 11:13:13 PM | Azure Portal | Success | 0 | Not Applied | Multifactor authentication |
| 11:13:10 PM | Azure Portal | Interrupted | 50140 | Not Applied | Multifactor authentication |
| 11:13:07 PM | Azure Portal | Interrupted | 50203 | Not Applied | Multifactor authentication |
| 11:12:52 PM | Azure Portal | Success | 0 | Not Applied | Single factor authentication |
| 9:26:37 PM | My Signins | Success | 0 | Not Applied | Multifactor authentication |
| 9:25:09 PM | My Signins | Interrupted | 50055 | Not Applied | Multifactor authentication |

Every break glass sign in shows Conditional Access "Not Applied." Two show a requirement of single factor, which means nothing in the tenant demanded MFA for that sign in. That is the tradeoff of excluding an emergency account from every policy, and it is why the account needs strong compensating controls (see "What I would change").
