# Onboarding Test Scenarios

Ten scenarios covering the account types and risks identified in [`01-migration-plan.md`](01-migration-plan.md). Each one tests the design in [`config/`](../config/) against an expected outcome.

## How tests were run

- **Tool:** AWS CLI `simulate-custom-policy`, run in CloudShell, evaluating each policy from `config/policies/` in isolation. The account-level simulator was unavailable: the account sits under an AWS Organizations guardrail that blocks S3 actions (`AllowedByOrganizations: false`), and the free plan blocks the console simulator. IAM user groups are also unavailable, so test users have policies attached directly.
- **Context keys:** set manually where needed (`aws:MultiFactorAuthPresent`, `aws:CurrentTime`, resource tags).
- **Walkthrough tests:** scenarios 5, 9 and 10 depend on process steps (group changes, offboarding), so they are verified by walking through the steps and checking the resulting access.
- **Data:** all users are fictional, from `data/users-sample.csv`.

## Summary

| # | Scenario | User(s) | Method | Result |
|---|---|---|---|---|
| 1 | Standard new hire, Finance | asmith | Custom policy simulation | Pass |
| 2 | HR hire with and without sensitive records access | nfoster, oadams | Simulator | |
| 3 | New IT admin and self-escalation attempt | sgarcia, sgarcia-admin | Custom policy simulation | Pass (1 limitation confirmed) |
**Totals:** 4 passed (scenarios 4 and 8 after fixes), 0 failing, 2 failed initially and fixed (scenarios 4 and 8), 6 not yet run. See [`04-policy-change-log.md`](04-policy-change-log.md) for policy changes.
| 5 | Department transfer (Sales to Finance) | tbrooks | Walkthrough + Simulator | |
| 6 | User who has not enrolled MFA | lbaker | Simulator | |
| 7 | Service account converted to role | svc-backup | Simulator | |
| 8 | Help desk password reset: regular vs. admin | ulee | Custom policy simulation | Pass (after fix) |
| 9 | Terminated employee and orphaned account | rdiaz, ibutler | Walkthrough | |
| 10 | Shared account retirement | shared-frontdesk | Walkthrough | |

**Totals:** 4 passed (scenarios 4 and 8 after fixes), 0 failing, 2 failed initially and fixed (scenarios 4 and 8), 6 not yet run. See [`04-policy-change-log.md`](04-policy-change-log.md) for policy changes.
---

## Scenario 1: Standard new hire, Finance
**User:** `asmith` (grp-all-staff, grp-finance-users)
**Expected**
- Read and write on S3 objects tagged `department=Finance`: allowed
- Read on objects tagged `department=HR`: denied
- Access to payroll bucket: denied
- Any `iam:*` action: denied
- Any action without MFA: denied

**Actual:** Tested `dept-readwrite.json` for a user tagged `department=Finance`, `privileged=false`:
- Read Finance/standard object: allowed
- Read HR/standard object: implicitDeny
- Read Finance/restricted object: implicitDeny
- Write Finance/standard object: allowed
- `iam:CreateUser`: implicitDeny

MFA enforcement is not part of this policy and is tested in scenario 6.
**Result:** ☒ Pass ☐ Fail

## Scenario 2: HR hire with and without sensitive records access
**Users:** `nfoster` (in grp-hr-records) and `oadams` (not in grp-hr-records)
**Expected**
- `nfoster`: access to HR records bucket allowed
- `oadams`: access to HR records bucket denied; general HR department data allowed

**Actual:**
**Result:** ☐ Pass ☐ Fail

## Scenario 3: New IT admin and self-escalation attempt
**Users:** `sgarcia` (daily identity) and `sgarcia-admin` (admin identity)
**Expected**
- `sgarcia`: no server administration rights
- `sgarcia-admin`: server administration allowed
- `sgarcia-admin` attempting `iam:AttachUserPolicy`, `iam:CreateAccessKey`, `iam:PutUserPermissionsBoundary`: **denied** by the permission boundary
- `sgarcia-admin` attempting `cloudtrail:StopLogging`: denied

**Actual:**

Daily identity (`sgarcia`, baseline policy only):
- ec2:StartInstances: implicitDeny (no server rights on the daily account)

Admin identity (`sgarcia-admin`, `it-sysadmin.json` + permission boundary):
- ec2:StartInstances: allowed
- iam:AttachUserPolicy: explicitDeny
- iam:CreateAccessKey: explicitDeny
- iam:PutUserPermissionsBoundary: explicitDeny
- cloudtrail:StopLogging: explicitDeny

Stress test: over-broad `Allow *` policy with and without the boundary:
- Without boundary: iam:AttachUserPolicy and cloudtrail:StopLogging both allowed
- With boundary: ec2:StartInstances allowed; the four escalation actions explicitDeny
- With boundary: `sso:CreateAccountAssignment` **allowed**. The IAM boundary does not cover Identity Center (risk R1). Confirmed limitation, not a policy change.

**Result:** ☒ Pass ☐ Fail (1 limitation confirmed)

## Scenario 4: Contractor before and after contract end
**User:** `sjenkins` (Sales, contract end 2026-12-31)
**Expected**
- Simulated date 2026-10-15: read Sales-tagged objects allowed; write denied; any `iam:*` denied
- Simulated date 2027-01-05: **all** actions denied
- Read of Finance-tagged objects at any date: denied

**Actual:**

First run (before fixes), `sjenkins` unless noted:
- 2026-10-15, read Sales/standard: allowed
- 2026-10-15, write an object: explicitDeny
- 2026-10-15, read Finance/standard: implicitDeny
- 2026-10-15, `iam:CreateUser`: explicitDeny
- 2026-10-15, read Sales/**restricted**: **allowed (unexpected)**
- 2027-01-05 (after contract end), read Sales/standard: explicitDeny
- `hmorgan`, 2026-12-01 (after his contract end), read Operations/standard: **allowed (unexpected)**
- `sjenkins`, 2026-12-01 (still under contract), read Sales/standard: allowed

Two failures, fixed as [policy change 2](04-policy-change-log.md) (restricted data) and policy change 3 (wrong date in `hmorgan`'s expiry file).

Retest after fixes:
- `sjenkins`, 2026-10-15, read Sales/standard: allowed
- `sjenkins`, 2026-10-15, read Sales/restricted: implicitDeny
- `hmorgan`, 2026-12-01, read Operations/standard: explicitDeny
- `sjenkins`, 2026-12-01, read Sales/standard: allowed

**Result:** ☒ Failed initially (2 issues), fixed, retested: Pass

**Not tested:** sessions already issued before the contract end date remain valid until they expire (up to 4 hours).
## Scenario 5: Department transfer (Sales to Finance)
**User:** `tbrooks` moves from Sales to Finance
**Steps:** remove from `grp-sales-users`, update `department` tag to Finance, add to `grp-finance-users`.
**Expected**
- After transfer: Finance data allowed, Sales data denied
- No overlap period where both are allowed
- Watch point: access in this design is driven by both group membership **and** the `department` tag. Check what happens if only one of the two is updated.

**Actual:**
**Result:** ☐ Pass ☐ Fail

## Scenario 6: User who has not enrolled MFA
**User:** `lbaker`
**Expected**
- `aws:MultiFactorAuthPresent = false`: MFA setup actions allowed; `s3:GetObject` denied
- `aws:MultiFactorAuthPresent = true`: normal Sales access allowed

**Actual:**
**Result:** ☐ Pass ☐ Fail

## Scenario 7: Service account converted to role
**Identity:** `svc-backup` replaced by `role-backup-service`
**Expected**
- Backup service assuming the role: allowed; read on backup sources and write to backup destination: allowed
- A human user assuming the role: denied (trust policy)
- Interactive console login: not possible
- `iam:CreateAccessKey` and `iam:PassRole` to non-approved roles: denied
- Broad rights from the old Backup Operators group: no longer present

**Actual:**
**Result:** ☐ Pass ☐ Fail

## Scenario 8: Help desk password reset: regular vs. admin
**User:** `ulee` (grp-it-helpdesk)
**Expected**
- Reset password for `lbaker` (tag `privileged=false`): allowed
- Reset password for `dpatel-admin` (tag `privileged=true`): denied
- Reset password for an untagged user: denied
- Changing a user's `privileged` tag: denied
- **Known risk (confirmed):** `permission-boundary.json` denied `iam:UpdateLoginProfile` for everyone, which blocked the help desk's resets. Fixed; see the change log.

**Actual:**

Help desk policy alone:
- Reset non-privileged user (`lbaker`, `privileged=false`): allowed
- Reset privileged user (`dpatel-admin`, `privileged=true`): explicitDeny
- Reset untagged user: implicitDeny
- Change a user's tag: explicitDeny

Help desk policy + permission boundary, **before the fix**:
- Reset non-privileged user: **explicitDeny (unexpected, expected allowed)**

Help desk policy + permission boundary, **after the fix** (see [policy change 1](04-policy-change-log.md)):
- Reset non-privileged user: allowed
- Reset privileged user: explicitDeny
- Reset untagged user: explicitDeny

**Result:** ☒ Failed initially, fixed, retested: Pass

## Scenario 9: Terminated employee and orphaned account
**Users:** `rdiaz` (terminated during test) and `ibutler` (orphaned: left 2026-06, account still active)
**Steps:** disable the identity, remove all group memberships, revoke active sessions.
**Expected**
- No group memberships and no permission sets remain
- New sign-in attempts fail
- Existing sessions end no later than the permission set's session length (8 hours for standard users); record this as a limitation
- `ibutler` is disabled first, data ownership reviewed, then deleted (per mapping file)

**Actual:**
**Result:** ☐ Pass ☐ Fail

## Scenario 10: Shared account retirement
**Account:** `shared-frontdesk`
**Expected**
- Not created in the target environment (mapping action: `retire`)
- Each person who used it gets an individual named account with MFA
- No identity named `shared-frontdesk` exists after cutover
- Activity afterward is traceable to named individuals in CloudTrail

**Actual:**
**Result:** ☐ Pass ☐ Fail

---

## Issues found
(Link each failure to a change in `04-policy-change-log.md`.)
