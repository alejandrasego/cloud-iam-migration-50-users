# Policy Change Log

## Change 1: Permission boundary blocked help desk password resets

**Found in:** Scenario 8 (help desk password reset)
**Date:** 2026-10-08
**Files changed:** `config/policies/permission-boundary.json`
**Risk register:** R7

### What failed
`it-helpdesk-limited.json` correctly allows password resets for users tagged `privileged=false`. Tested alone, it behaved as designed. Tested with `permission-boundary.json` attached, the same reset was denied.

| Test | Policies evaluated | Expected | Result |
|---|---|---|---|
| A1. Reset non-privileged user | help desk | allowed | allowed |
| A2. Reset privileged user | help desk | explicitDeny | explicitDeny |
| A3. Reset untagged user | help desk | implicitDeny | implicitDeny |
| A4. Change a user's tag | help desk | explicitDeny | explicitDeny |
| **B1. Reset non-privileged user** | **help desk + boundary** | **allowed** | **explicitDeny (FAIL)** |

### Why it was a problem
The boundary listed `iam:UpdateLoginProfile` in `DenyCreatingBackdoorCredentials`, which denies it for every identity. A boundary caps all permissions, so the help desk could never reset a password and could not do its job. The two policies were each correct alone and wrong together.

### Change made
Removed `iam:UpdateLoginProfile` from the blanket deny and added a conditional deny that blocks resets on any target not explicitly tagged `privileged=false`:

```diff
   "Action": [
     "iam:CreateUser",
     "iam:CreateAccessKey",
     "iam:CreateLoginProfile",
-    "iam:UpdateLoginProfile",
     "iam:AddUserToGroup",
     "iam:UpdateAssumeRolePolicy"
   ],
+  {
+    "Sid": "DenyPasswordResetExceptNonPrivilegedTargets",
+    "Effect": "Deny",
+    "Action": "iam:UpdateLoginProfile",
+    "Resource": "*",
+    "Condition": {
+      "StringNotEquals": { "aws:ResourceTag/privileged": "false" }
+    }
+  }
```

### Retest (help desk + boundary)
| Test | Expected | Result |
|---|---|---|
| B1. Reset non-privileged user | allowed | allowed |
| B2. Reset privileged user | explicitDeny | explicitDeny |
| B3. Reset untagged user | deny | explicitDeny |

### Residual risk
- Any permission set that grants `iam:UpdateLoginProfile` can now use it on accounts tagged `privileged=false`. Only the help desk is granted it.
- Anyone able to edit a user's `privileged` tag could defeat the control. Tag edits are denied in the help desk policy and should be restricted for every other permission set too (open item).
- Tested with simulated tag values via `simulate-custom-policy`; not verified against live users.

---

## Change 2: Contractor policy allowed reads of restricted data

**Found in:** Scenario 4 (contractor before and after contract end)
**Date:** 2026-10-08
**Files changed:** `config/policies/contractor-limited.json`
**Risk register:** R10

### What failed
A contractor (`sjenkins`, Sales) could read Sales objects tagged `sensitivity=restricted`.

| Test | Expected | Result |
|---|---|---|
| Read Sales/standard object | allowed | allowed |
| Read Sales/**restricted** object | deny | **allowed (FAIL)** |

### Why it was a problem
When the `sensitivity=standard` condition was added to `dept-readwrite.json`, `contractor-limited.json` was not updated to match. Employees were blocked from restricted data, but the read-only contractor policy still matched on department alone. A contractor could therefore read data that regular employees in the same department could not.

### Change made
```diff
 "StringEquals": {
-  "s3:ExistingObjectTag/department": "${aws:PrincipalTag/department}"
+  "s3:ExistingObjectTag/department": "${aws:PrincipalTag/department}",
+  "s3:ExistingObjectTag/sensitivity": "standard"
 }
```

### Retest
| Test | Expected | Result |
|---|---|---|
| Read Sales/standard object | allowed | allowed |
| Read Sales/restricted object | deny | implicitDeny |

### Residual risk
The same rule now lives in two policy files, so they can drift apart again. A shared policy fragment, or an automated check that every policy applies the sensitivity rule, would prevent that (see write-up, next improvements).

---

## Change 3: Wrong expiry date in a contractor's policy file

**Found in:** Scenario 4
**Date:** 2026-10-08
**Files changed:** `config/policies/contractor-expiry-hmorgan.json`
**Risk register:** R11

### What failed
`hmorgan` (contract end 2026-11-30 per `data/users-sample.csv`) was still allowed on 2026-12-01.

| Test | Expected | Result |
|---|---|---|
| hmorgan reads Operations/standard on 2026-12-01 | explicitDeny | **allowed (FAIL)** |
| sjenkins reads Sales/standard on 2026-12-01 | allowed | allowed |

### Why it was a problem
The expiry policy logic was correct, but `hmorgan`'s file contained `sjenkins`' contract end date (`2026-12-31`), most likely from copying one file to create the other. His access would have lingered for a month past his contract.

### Change made
```diff
-"DateGreaterThan": { "aws:CurrentTime": "2026-12-31T23:59:59Z" }
+"DateGreaterThan": { "aws:CurrentTime": "2026-11-30T23:59:59Z" }
```

### Retest
| Test | Expected | Result |
|---|---|---|
| hmorgan reads Operations/standard on 2026-12-01 | explicitDeny | explicitDeny |
| sjenkins reads Sales/standard on 2026-12-01 | allowed | allowed |

The dates in both expiry files were also checked against the inventory CSV and match.

### Residual risk
Per-contractor hardcoded dates are error-prone, as this change shows. Generating expiry policies from the inventory, or adding a check that compares them to `contract_end`, would remove this class of mistake.
