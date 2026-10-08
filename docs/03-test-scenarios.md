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
| 3 | New IT admin and self-escalation attempt | sgarcia, sgarcia-admin | Simulator | |
| 4 | Contractor before and after contract end | sjenkins | Simulator | |
| 5 | Department transfer (Sales to Finance) | tbrooks | Walkthrough + Simulator | |
| 6 | User who has not enrolled MFA | lbaker | Simulator | |
| 7 | Service account converted to role | svc-backup | Simulator | |
| 8 | Help desk password reset: regular vs. admin | ulee | Simulator | |
| 9 | Terminated employee and orphaned account | rdiaz, ibutler | Walkthrough | |
| 10 | Shared account retirement | shared-frontdesk | Walkthrough | |

**Totals:** ___ passed, ___ failed, ___ fixed and retested. See [`04-policy-change-log.md`](04-policy-change-log.md) for any policy changes.

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
**Result:** ☐ Pass ☐ Fail

## Scenario 4: Contractor before and after contract end
**User:** `sjenkins` (Sales, contract end 2026-12-31)
**Expected**
- Simulated date 2026-10-15: read Sales-tagged objects allowed; write denied; any `iam:*` denied
- Simulated date 2027-01-05: **all** actions denied
- Read of Finance-tagged objects at any date: denied

**Actual:**
**Result:** ☐ Pass ☐ Fail

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
- **Known risk to check:** `permission-boundary.json` denies `iam:UpdateLoginProfile` for everyone. If the boundary is attached to the help desk role, the first expected result may fail. Record exactly what the simulator shows.

**Actual:**
**Result:** ☐ Pass ☐ Fail

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
