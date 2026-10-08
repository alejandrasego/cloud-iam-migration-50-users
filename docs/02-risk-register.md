# Privilege Escalation Risk Register

Risks evaluated against the design in [`config/`](../config/). "Status" reflects whether the mitigation has been verified in testing ([`03-test-scenarios.md`](03-test-scenarios.md)); update it as results come in.

**Scale:** Likelihood and Impact are Low / Medium / High, assessed before mitigation.

| ID | Risk | How it happens | L | I | Mitigation | Where implemented | Framework | Status |
|---|---|---|---|---|---|---|---|---|
| R1 | Self-granted admin via policy edits | User with IAM write rights creates or attaches a policy giving themselves admin | M | H | Deny policy create/attach/put actions; no admin permission set has IAM write | `permission-boundary.json` | NIST AC-6, CIS 6 | Verified for IAM actions in scenario 3 (AttachUserPolicy denied). Identity Center gap confirmed (sso:CreateAccountAssignment allowed); open. |
| R2 | Boundary removal | User deletes or replaces their own permission boundary | M | H | Deny `Put/DeletePermissionsBoundary` actions | `permission-boundary.json` | NIST AC-6 | Verified in scenario 3 (PutUserPermissionsBoundary denied). |
| R3 | Backdoor credentials | User creates a new IAM user, access key, or login profile for themselves or another account | M | H | Deny `CreateUser`, `CreateAccessKey`, `CreateLoginProfile` | `permission-boundary.json` | CIS 5, NIST AC-2 | Partly verified in scenario 3 (CreateAccessKey denied); CreateUser and CreateLoginProfile not yet tested. |
| R4 | Role created without a ceiling | User creates a new role with no boundary, then assumes it | M | H | `CreateRole` allowed only with the approved boundary attached | `permission-boundary.json` | NIST AC-6 | Not yet tested |
| R5 | PassRole abuse | User passes a powerful role to a service they control | M | H | `PassRole` denied except to three approved service roles | `permission-boundary.json` | AWS Well-Architected SEC03 | Not yet tested |
| R6 | Trust policy rewrite | User edits a role's trust policy so they can assume it | L | H | Deny `UpdateAssumeRolePolicy` | `permission-boundary.json` | NIST AC-6 | Not yet tested |
| R7 | Help desk takes over an admin account | Help desk resets an admin's password/MFA, or edits the `privileged` tag first | M | H | Allow resets only on `privileged=false`; explicit deny on `true`; tag edits denied | `it-helpdesk-limited.json` | NIST AC-5, AC-6 | Failure found and fixed in scenario 8; see [policy change log](04-policy-change-log.md)
| R8 | Stolen password alone gives access | Credential theft or phishing | H | H | MFA required for all access | `mfa-enforcement.json`, Identity Center MFA setting | NIST 800-63B | Not yet tested (S6) |
| R9 | Audit log tampering | Admin stops or edits CloudTrail to hide activity | L | H | Deny `StopLogging`, `DeleteTrail`, `UpdateTrail` | `permission-boundary.json` | NIST AU-9, CIS 8 | Verified in scenario 3 (StopLogging denied). |
| R10 | Cross-department data access | User reads another department's or restricted data | H | M | Tag-based access (`department`, `sensitivity=standard`); fails closed on missing tags | `dept-readwrite.json` | NIST AC-3, AC-6 | Not yet tested (S1, S2) |
| R11 | Contractor access outlives contract | No one offboards the account on time | H | M | Per-contractor date-based deny | `contractor-expiry-*.json` | NIST AC-2(3) | Not yet tested (S4) |
| R12 | Stale access after transfer | Old department group or tag left in place | H | M | Transfer procedure: remove old access first, then add new | `groups-and-roles.md` section 6 | CIS 6.2 | Not yet tested (S5) |
| R13 | Shared and orphaned accounts | No accountability; ex-employee account still active | H | M | Retire shared account; disable then delete orphaned account | `user-group-mapping.csv` | CIS 5.3 | Not yet tested (S9, S10) |
| R14 | Over-privileged service account | `svc-backup` holds Backup Operators rights; credentials leak | M | H | Convert to scoped roles; no interactive login; no long-lived keys | `groups-and-roles.md` section 4 | CIS 5.4 | Not yet tested (S7) |
| R15 | Daily account is also an admin account | Phishing a daily-use account yields admin | H | H | Separate `-admin` identities with 1-hour sessions | `user-group-mapping.csv`, `groups-and-roles.md` | NIST AC-6(5) | Design only |
| R16 | Social engineering of help desk | Attacker phones in pretending to be a user and gets an MFA reset | M | H | Process control: identity verification (manager approval or call-back) before any MFA reset | Process, not technical | NIST IA-5 | Documented, not technically enforceable |

## Known open issues

**Help desk policy vs. permission boundary (R7): resolved.** The boundary's blanket deny on `iam:UpdateLoginProfile` blocked legitimate help desk resets. Fixed with a conditional deny and retested; see [`04-policy-change-log.md`](04-policy-change-log.md).

**Identity Center is not covered by the IAM boundary (R1).** Permission sets are managed through `sso-admin:*` actions, which the IAM boundary does not restrict. An identity admin could assign broad access to themselves. Recommended mitigation: an SCP limiting account assignments, plus alerting on assignment events. Not implemented.

## Residual risk summary

| Risk area | Remaining exposure |
|---|---|
| Identity Center privilege assignment | High until SCP and alerting are in place |
| Help desk social engineering | Medium; depends on process discipline |
| Session persistence after offboarding | Low to medium; sessions last up to 8 hours |
