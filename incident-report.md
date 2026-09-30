# Incident Report: Simulated IAM Persistence and Defense Evasion

**Type:** Simulated attack (authorized, in my own AWS account)
**Date:** September 30, 2026 (UTC)
**Analyst:** Tirth Patel
**Status:** Closed — contained and remediated

---

## 1. Summary

An attacker using the compromised `tirth-admin` identity created a backdoor IAM user with full admin rights, generated long-term credentials for it, and then disabled CloudTrail logging to hide further activity. The attacker also signed in as the root user.

The detection pipeline alerted on the persistence and log-tampering actions within seconds. The root sign-in was recorded but not alerted, because it was logged in a different region from the detection rule.

---

## 2. Timeline (UTC)

| Time | Event | Actor | Source IP | Region | Alert |
|---|---|---|---|---|---|
| 05:32:46 | ConsoleLogin | tirth-admin | `<REDACTED>` | us-east-2 | — (not monitored) |
| 05:55:39 | **CreateUser** `backdoor-admin` | tirth-admin | `<REDACTED>` | us-east-1 | ✅ |
| 05:55:40 | **AttachUserPolicy** `AdministratorAccess` → backdoor-admin | tirth-admin | `<REDACTED>` | us-east-1 | ✅ |
| 05:58:45 | **CreateAccessKey** for backdoor-admin | tirth-admin | `<REDACTED>` | us-east-1 | ✅ |
| 06:01:28 | **StopLogging** on `security-baseline-trail` | tirth-admin | `<REDACTED>` | us-east-1 | ✅ |
| 06:04:10 | **ConsoleLogin** as root | root | `<REDACTED>` | us-east-2 | ❌ |

---

## 3. Indicators

- **One identity, one IP** for the entire attack chain → a single compromised credential
- **`mfaAuthenticated: false`** on every attacker event → the session had no MFA
- **Classic persistence pattern:** new user → admin policy → access key within ~3 minutes
- **Defense evasion:** logging stopped ~3 minutes after the backdoor was complete

---

## 4. Analysis

| Stage | Action | MITRE ATT&CK |
|---|---|---|
| Persistence | Created a new IAM user | T1136.003 Create Account: Cloud Account |
| Privilege escalation | Attached AdministratorAccess | T1098 Account Manipulation |
| Persistence | Created an access key for the backdoor | T1098.001 Additional Cloud Credentials |
| Defense evasion | Stopped CloudTrail logging | T1562.008 Disable or Modify Cloud Logs |
| Privileged access | Root console sign-in | T1078.004 Valid Accounts: Cloud Accounts |

The attacker's goal was to keep access even if the original credential was revoked, then reduce visibility. Because `StopLogging` is itself recorded, the tampering was caught before logging went dark.

---

## 5. Response

**Containment**
- Restarted CloudTrail logging

**Eradication**
- Deactivated and deleted the backdoor access key
- Deleted the `backdoor-admin` user

**Recovery**
- Verified only the legitimate `tirth-admin` user remains
- Verified all 3 detection rules remain enabled

---

## 6. Detection Gap

**Gap:** Root sign-in was not alerted.
**Root cause:** Console sign-in events for this account were recorded in us-east-2, while the `root-login` rule ran only in us-east-1. EventBridge rules only see events in their own region.
**Action taken:** Added a us-east-2 rule to forward root sign-in events to us-east-1. Alert delivery is still unconfirmed and under investigation.

---

## 7. Recommendations

1. **Enforce MFA** on all IAM users (tirth-admin had none)
2. **Enable log file validation** on the trail (found disabled)
3. **Deploy detection rules to every region** (e.g. CloudFormation StackSets)
4. **Replace long-term IAM users** with IAM Identity Center and temporary credentials
5. **Add auto-remediation** — e.g. a Lambda that restarts logging on StopLogging
