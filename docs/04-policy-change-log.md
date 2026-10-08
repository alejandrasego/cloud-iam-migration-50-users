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
