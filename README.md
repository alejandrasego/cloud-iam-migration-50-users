# Cloud IAM Migration: 50 Local Accounts to AWS

A migration plan, configuration, and test record for moving 50 local user accounts to a cloud IAM environment, built as a project-administration portfolio piece. All users and data are fictional.

This is a design, configuration, and policy-testing project; nothing here is deployed to a live environment.

**Author:** Alejandra Segoviano | **Role targeted:** Identity & Access Management, Cloud Identity Administration

## Results at a glance

| | |
|---|---|
| Accounts planned | 50 (45 migrate, 3 become service roles, 1 retired, 1 disabled then deleted) plus 4 new separate admin identities |
| Test scenarios | **10 of 10 pass** (3 passed only after a fix) |
| Failures found in testing | 4 issues across 3 scenarios (4, 5, 8) |
| Documented changes | 4 (3 policy fixes, 1 procedure fix), see the [change log](docs/04-policy-change-log.md) |
| Frameworks referenced | NIST SP 800-53 (AC, AU, IA, PS), NIST SP 800-63B, CIS Controls v8, AWS Well-Architected Security Pillar |

## The headline finding
Two policies that each passed testing alone **failed together**: the permission boundary's blanket deny on password resets blocked the help desk from doing its job (scenario 8, [change 1](docs/04-policy-change-log.md)). It was found by testing the help desk policy with and without the boundary, fixed with a conditional deny, and retested. The other three failures (a contractor reading restricted data, a wrong expiry date copied between contractor files, and a transfer procedure that left old access in place) are in the same log.

## Where to look

| Document | What it covers |
|---|---|
| [`docs/01-migration-plan.md`](docs/01-migration-plan.md) | Scope, RACI, phases, communications, rollback |
| [`docs/02-risk-register.md`](docs/02-risk-register.md) | 17 privilege escalation and access risks, mitigations, status |
| [`docs/03-test-scenarios.md`](docs/03-test-scenarios.md) | 10 onboarding scenarios with results |
| [`docs/04-policy-change-log.md`](docs/04-policy-change-log.md) | Each failure: what broke, why, the fix, the retest |
| [`docs/05-writeup.md`](docs/05-writeup.md) | Failure cases, remaining limitations, next improvements |
| [`config/groups-and-roles.md`](config/groups-and-roles.md) | Groups, permission sets, guardrails, lifecycle rules |
| [`config/policies/`](config/policies/) | The IAM policies (JSON) |
| [`config/user-group-mapping.csv`](config/user-group-mapping.csv) | How each of the 50 accounts is handled |
| [`data/users-sample.csv`](data/users-sample.csv) | The fictional inventory |
| [`docs/06-test-commands.md`](docs/06-test-commands.md) | The exact commands and raw output for every scenario |

## How it was tested
Policies were evaluated with the AWS CLI's `simulate-custom-policy` in CloudShell, using simulated tag and MFA values. The account-level simulator was unusable: for the S3 actions tested it returned `explicitDeny` with no matching policy statement and `AllowedByOrganizations: false`, which points to an account-level restriction rather than a flaw in the policies (the write-up explains how that was identified). Scenario 9 combines a policy test with an inventory check, scenario 10 is an inventory check, and scenario 7's trust policy was reviewed by inspection rather than simulated.**

## Known limitations (summary)
- The permission boundary does not cover Identity Center actions (`sso:*`); confirmed in testing.
- MFA for staff is enforced at the Identity Center layer, which could not be exercised here. The IAM policy is a backstop.
- Contractor expiry uses hardcoded per-person dates and does not end sessions already issued.
- Session revocation covers temporary credentials only.
- Users of the shared front desk account are not identified in the inventory.
- IAM groups were unavailable on the free plan, so group membership was modeled by policy attachment.

Full list and next improvements: [`docs/05-writeup.md`](docs/05-writeup.md).

## Notes
- Account ID `123456789012` and bucket names are placeholders. Replace them before any real deployment.
- Licensed under MIT.
