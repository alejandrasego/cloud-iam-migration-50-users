# Groups and Permission Sets

Design for the 50-user migration to AWS IAM Identity Center. Source inventory: [`data/users-sample.csv`](../data/users-sample.csv).

## 1. Design rules

1. **No permissions on individual users.** Users get access only through group membership.
2. **Groups map to permission sets.** A permission set is the bundle of access a job role needs.
3. **Department data is separated with tags.** Each S3 object carries a department tag and a sensitivity tag (standard or restricted). The department baseline only allows standard data; restricted data requires a dedicated permission set.
4. **Admin access is separate.** Privileged staff get a second identity (`<username>-admin`) used only for admin tasks.
5. **Every permission set requires MFA** and has a defined session length.

## 2. Groups

| Group | Purpose | Members (from inventory) |
|---|---|---|
| `grp-all-staff` | Baseline: self-service MFA and password only | All human identities (49: 45 people plus 4 admin identities) |
| `grp-finance-users` | Finance department data | jlopez, asmith, bjohnson, cmartinez, dkim, evargas, fobrien, gnakamura |
| `grp-finance-payroll` | Payroll data (sensitive) | dkim |
| `grp-finance-audit` | Read-only audit access to finance data | hrossi |
| `grp-hr-users` | HR department data | mchen, nfoster, oadams, pwright, qhughes, rdiaz |
| `grp-hr-records` | Employee records (sensitive) | mchen, nfoster, pwright |
| `grp-it-helpdesk` | Limited user support | ulee, vclark |
| `grp-it-security` | Security monitoring, read-only | wlewis |
| `grp-it-developers` | Dev environment only | ycollins |
| `grp-it-sysadmin` | Server administration | sgarcia-admin |
| `grp-it-network` | Network administration | tmoore-admin |
| `grp-it-dba` | Database administration | xwalker-admin |
| `grp-it-identity-admin` | Manage users and groups | dpatel-admin |
| `grp-sales-users` | Sales department data | kwilliams, lbaker, mcooper, nreed, obailey, pcox, qward, rhoward, tbrooks, uperry, vgray |
| `grp-sales-crm-admin` | CRM administration | kwilliams |
| `grp-ops-users` | Operations department data | rnguyen, asanders, bpeterson, cramirez, dwatson, ebennett, fcastillo, gflores, jsimmons |
| `grp-ops-procurement` | Procurement data | bpeterson |
| `grp-contractors` | Time-limited, restricted access | sjenkins, hmorgan |

**Not placed in groups:** `svc-payroll`, `svc-backup`, `svc-reporting` (become roles, see section 4), `shared-frontdesk` (retired), `ibutler` (orphaned, disabled and deleted).

## 3. Permission sets

| Permission set | Assigned to group(s) | Grants | Session | MFA |
|---|---|---|---|---|
| `Baseline-SelfService` | grp-all-staff | Manage own MFA device and password; nothing else | 8 hrs | Yes |
| `Dept-ReadWrite` | grp-finance-users, grp-hr-users, grp-sales-users, grp-ops-users | Read/write S3 objects where bucket tag `department` equals the user's department | 8 hrs | Yes |
| `Finance-Payroll` | grp-finance-payroll | Read/write payroll bucket | 4 hrs | Yes |
| `Finance-AuditReadOnly` | grp-finance-audit | Read-only on finance buckets and logs | 4 hrs | Yes |
| `HR-Records` | grp-hr-records | Read/write HR records bucket | 4 hrs | Yes |
| `Ops-Procurement` | grp-ops-procurement | Read/write procurement bucket | 8 hrs | Yes |
| `CRM-Admin` | grp-sales-crm-admin | Admin on CRM application resources only | 4 hrs | Yes |
| `IT-HelpDesk-Limited` | grp-it-helpdesk | Reset passwords and MFA for **non-privileged users only**; cannot touch admin identities | 4 hrs | Yes |
| `IT-Security-ReadOnly` | grp-it-security | Read-only security audit access; view CloudTrail and IAM configuration | 8 hrs | Yes |
| `IT-Developer` | grp-it-developers | Power-user access in the dev account only; no IAM actions | 8 hrs | Yes |
| `IT-SysAdmin` | grp-it-sysadmin | Manage servers; **no IAM write access** | 1 hr | Yes |
| `IT-NetworkAdmin` | grp-it-network | Manage network resources; no IAM write access | 1 hr | Yes |
| `IT-DBAdmin` | grp-it-dba | Manage databases; no IAM write access | 1 hr | Yes |
| `IT-IdentityAdmin` | grp-it-identity-admin | Create users and groups, assign permission sets; **cannot modify guardrails or its own permissions** | 1 hr | Yes |
| `Contractor-Limited` | grp-contractors | Read-only on their department's data; capped by permission boundary | 4 hrs | Yes |

## 4. Service accounts converted to roles

| Original | New role | Trust | Permissions | Interactive login |
|---|---|---|---|---|
| `svc-payroll` | `role-payroll-service` | Payroll application only | Read/write payroll bucket | Denied |
| `svc-backup` | `role-backup-service` | Backup service only | Read-only on backup source resources plus write to the backup destination; **replaces Backup Operators rights** | Denied |
| `svc-reporting` | `role-reporting-service` | Reporting service only | Read-only on reporting database | Denied |

## 5. Guardrails (apply to everyone, including admins)

| Guardrail | Mechanism | Prevents |
|---|---|---|
| Deny all actions without MFA | Service control policy (SCP) / policy condition | Access with stolen passwords alone |
| Deny disabling or deleting CloudTrail | SCP | Covering tracks |
| Deny creating IAM users and long-lived access keys | SCP, except `IT-IdentityAdmin` | Backdoor credentials |
| Deny attaching `AdministratorAccess` or editing own policies | Permission boundary | Self-granted admin (privilege escalation) |
| Deny `iam:PassRole` except to approved roles | Policy condition | Passing a powerful role to a resource the user controls |
| Deny root user actions | SCP | Root account misuse |
| Contractor access ends on contract date | Expiry date tag plus scheduled review | Lingering contractor access |

## 6. Lifecycle rules

- **New hire:** user created in Identity Center, added to department group, MFA enrolled at first sign-in.
- **Transfer:** remove from old department groups **first**, then add to new ones. Never leave both.
- **Termination:** disable immediately, remove group memberships, delete after the retention period.
- **Contractor:** access end date set at creation; reviewed 7 days before expiry.
- **Access review:** managers confirm group membership quarterly.
