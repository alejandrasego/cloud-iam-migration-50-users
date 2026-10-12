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

---

## Change 4: Transfer procedure left old-department access in place (design/procedure change)

**Found in:** Scenario 5 (department transfer)
**Date:** 2026-10-12
**Files changed:** `config/groups-and-roles.md` (section 6). No policy JSON changed.
**Risk register:** R12

### What failed
When a transfer changed the user's groups but not their `department` tag, the user kept reading their old department's data.

| Case | Situation | Expected | Result |
|---|---|---|---|
| A1 / A2 | Before transfer (tag Sales) | allowed / implicitDeny | as expected |
| B1 / B2 | Clean transfer (tag Finance) | allowed / implicitDeny | as expected |
| C1 | Group changed, tag still Sales: new Finance data | implicitDeny | implicitDeny |
| **C2** | **Group changed, tag still Sales: old Sales data** | **deny** | **allowed (FAIL)** |
| D1 | Tag changed, old group left: old Sales data | implicitDeny | implicitDeny |
| E1 | Tag changed, stale HR-records group left: HR restricted data | implicitDeny | implicitDeny |

### Why it was a problem
Department data access is driven by the `department` tag. The department groups all map to the same permission set, so group changes do not affect it. The documented procedure (remove old groups first, then add new ones) put the wrong control first. A transfer that stopped after the group steps left the old tag, and with it the old access.

### Change made
Transfer procedure now starts with the tag:

```diff
-- **Transfer:** remove from old department groups **first**, then add to new ones. Never leave both.
+- **Transfer:** update the user's `department` tag **first**, because data access follows the tag, not the group. Then remove the old department groups and add the new ones. Never rely on group changes alone.
```

### Verification
With the tag changed first, no intermediate state leaves old access: D1 shows old Sales data is denied once the tag is Finance, even with the old group still in place, and E1 shows a stale HR-records group grants nothing after the tag changes. These are the results of the original run, which already covered the corrected order; no policy was altered.

### Residual risk
- The procedure depends on a person following it. A periodic check comparing each user's tag to their groups would catch missed steps (not implemented).
- Tag edits are powerful: anyone who can edit a user's `department` tag can change that user's data access. Tag edits are denied to the help desk and should be limited to identity admins.
- A changed tag takes effect when the user's next session starts, so access can persist until an existing session expires (not tested).
- In production, the department should come from the identity source (directory or HR) as an Identity Center attribute, so a single record drives both tag and groups.
