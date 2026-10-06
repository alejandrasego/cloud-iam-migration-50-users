# Cloud IAM Migration Plan: 50 Local Accounts

**Author:** Alejandra Segoviano, Project Administrator
**Status:** Draft v1.0
**Target platform:** AWS (IAM Identity Center + IAM)
**Source data:** [`data/users-sample.csv`](../data/users-sample.csv) (fictional)

## 1. Purpose and objectives

Move 50 locally managed user accounts to a centrally managed cloud IAM environment with role-based access, enforced MFA, and no standing excess privilege.

**Objectives**
1. Migrate all 50 accounts with no loss of legitimate access.
2. Enforce MFA for 100% of human users.
3. Replace per-user permissions with group/role-based access (least privilege).
4. Remove or remediate risky legacy accounts (orphaned, shared, over-privileged).
5. Produce evidence (test results, risk register) that the design works.

## 2. Scope

**In scope:** all 50 accounts in the inventory, group and role design, MFA, offboarding and contractor expiry rules, testing, rollback.

**Out of scope:** endpoint management, application-level permissions inside third-party SaaS tools, network redesign, HR system integration (listed as a future improvement).

## 3. Current state (from inventory)

| Item | Count |
|---|---|
| Total accounts | 50 |
| Employees | 43 |
| Contractors (with end dates) | 2 |
| Service accounts | 3 |
| Shared accounts | 1 |
| Orphaned accounts (user left, account active) | 1 |
| Accounts flagged privileged | 5 |
| Accounts with MFA enabled | 0 |

**Departments:** Finance 10, HR 6, IT 10, Sales 12, Operations 12.

**Known issues found in the inventory**
- No MFA on any account.
- IT Manager uses one account for both daily work and domain admin.
- `svc-backup` holds Backup Operators rights and is highly privileged.
- `shared-frontdesk` uses a shared password, so there is no individual accountability.
- `ibutler` left the company but the account is still active.
- Contractor accounts (`sjenkins`, `hmorgan`) have no automatic expiry.
- Help desk staff can reset passwords with no restriction on which accounts.

## 4. Target state design

| Principle | Decision |
|---|---|
| Identity source | AWS IAM Identity Center as the single place users sign in |
| Access model | Users belong to groups; groups map to permission sets; no permissions are assigned to individual users |
| Least privilege | Each permission set grants only what the department's job needs |
| Privileged access | Separate admin identities (e.g., `dpatel` and `dpatel-admin`); admin roles are assumed only when needed |
| MFA | Required for every human user at sign-in |
| Service accounts | Replaced with IAM roles where possible; no interactive login; no long-lived access keys |
| Guardrails | Service control policies and permission boundaries cap what anyone, including admins, can do |
| Logging | CloudTrail enabled for all IAM activity |

## 5. Account handling by type

| Account type | Count | Migration approach |
|---|---|---|
| Employee | 43 | Create user, assign to department group, enroll MFA |
| of which privileged | 4 | Create standard user plus separate admin role; admin requires MFA and is reviewed quarterly |
| Contractor | 2 | Create user with access that expires on the `contract_end` date |
| Service account | 3 | Convert to IAM roles or restricted identities; deny interactive login; rotate credentials |
| Shared account | 1 | Retire; replace with individual named accounts |
| Orphaned account | 1 | Do not migrate; disable, review for data ownership, then delete |

## 6. Phased timeline

| Phase | Week | Activities | Exit criteria |
|---|---|---|---|
| 0. Discovery | 1 | Validate inventory, confirm owners, map access to roles | Inventory signed off |
| 1. Design | 1-2 | Define groups, permission sets, SCPs, MFA policy | Design reviewed and approved |
| 2. Build | 2 | Configure the environment, load groups and policies | Config committed to repo |
| 3. Pilot | 3 | Migrate 5 users (one per department, plus 1 admin) | All pilot tests pass |
| 4. Wave 1 | 4 | Migrate 20 users (non-privileged) | Tests pass; no open critical issues |
| 5. Wave 2 | 5 | Migrate the remaining 25, including admins, contractors, service accounts | Tests pass |
| 6. Cutover | 6 | Disable local accounts; enforce cloud sign-in | No sign-in failures for 48 hours |
| 7. Decommission | 7-8 | Remove local accounts, final access review, document lessons learned | Sign-off |

## 7. RACI

| Activity | Project Administrator | IT Manager | Security Analyst | Dept. Managers | HR |
|---|---|---|---|---|---|
| Inventory validation | R | A | C | C | C |
| Role and policy design | C | A | R | C | I |
| Build and configuration | C | A | R | I | I |
| Testing | R | A | C | C | I |
| User communications | R | C | I | A | C |
| Cutover | R | A | C | I | I |
| Access review sign-off | C | R | C | A | I |

*R = Responsible, A = Accountable, C = Consulted, I = Informed.*

## 8. Communication plan

| When | Audience | Message |
|---|---|---|
| 2 weeks before wave | Affected users | What's changing, when, what they need to do |
| 1 week before | Affected users | MFA enrollment instructions |
| Day of migration | Affected users, help desk | Go-live notice, support contact |
| 48 hours after | Dept. managers | Confirmation and issue summary |

## 9. Risk and rollback

**Rollback approach:** local accounts stay active but disabled-ready until the end of Phase 6. If a wave fails its exit criteria, affected users revert to local sign-in and the cloud changes for that wave are undone.

**Rollback triggers:** more than 10% of wave users unable to sign in, any confirmed unauthorized access, or loss of access to a critical business system.

Privilege escalation risks are tracked separately in [`02-risk-register.md`](02-risk-register.md).

## 10. Framework alignment

| Control area | Reference |
|---|---|
| Account management | NIST SP 800-53 AC-2; CIS Control 5 |
| Least privilege | NIST SP 800-53 AC-6; CIS Control 6 |
| Multi-factor authentication | NIST SP 800-63B; CIS Control 6 |
| Separation of duties | NIST SP 800-53 AC-5 |
| Audit logging | NIST SP 800-53 AU-2; CIS Control 8 |
| Cloud best practice | AWS Well-Architected, Security Pillar (Identity and Access Management) |

## 11. Success criteria

- 50 of 50 accounts accounted for (migrated, retired, or deleted with a documented reason)
- 100% MFA coverage for human users
- 10 of 10 onboarding test scenarios documented with results
- Zero shared accounts and zero orphaned accounts after cutover
- No standing admin rights on daily-use accounts

## 12. Assumptions and open items

- Inventory is accurate as of the start of Phase 0.
- Business owners are available to approve role mappings.
- Application owners confirm which systems depend on the three service accounts before conversion.
- Open: decision on hardware keys vs. authenticator apps for admin MFA.
