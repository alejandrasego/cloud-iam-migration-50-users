# Test Commands and Raw Results

Every scenario in [`03-test-scenarios.md`](03-test-scenarios.md) was run from AWS CloudShell using `aws iam simulate-custom-policy`, which evaluates the policy files in [`config/policies/`](../config/policies/) with simulated context values (tags, MFA, time). This file shows the commands and the raw output so each result can be checked or rerun.

**Reading the results**
- `allowed`: a policy permits the action.
- `implicitDeny`: nothing permits it (the default).
- `explicitDeny`: a policy denies it outright. A deny always overrides an allow.
- Account ID `123456789012` and bucket names are placeholders. Output is shown without CloudShell prompts.
- Helper functions are redefined in each scenario so each section can be run on its own.

**Common setup** (run once per CloudShell session):

```bash
export AWS_PAGER=""
git clone https://github.com/alejandrasego/cloud-iam-migration-50-users.git
cd cloud-iam-migration-50-users/config/policies
```

---

## 0. Why the account-level simulator was not used

My first runs used `simulate-principal-policy` against a real test user (`asmith`, tags `department=Finance`, `privileged=false`, only `dept-readwrite` attached, no permissions boundary):

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/asmith \
  --action-names s3:GetObject \
  --resource-arns "arn:aws:s3:::example-bucket/*" \
  --context-entries \
    "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=Finance,ContextKeyType=string" \
    "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Finance,ContextKeyType=string" \
    "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string" \
  --output json
```

Output:

```json
{
    "EvaluationResults": [
        {
            "EvalActionName": "s3:GetObject",
            "EvalResourceName": "arn:aws:s3:::example-bucket/*",
            "EvalDecision": "explicitDeny",
            "MatchedStatements": [],
            "MissingContextValues": [],
            "OrganizationsDecisionDetail": {
                "AllowedByOrganizations": false
            },
            "EvalDecisionDetails": {},
            "ResourceSpecificResults": [
                {
                    "EvalResourceName": "arn:aws:s3:::example-bucket/*",
                    "EvalResourceDecision": "explicitDeny"
                }
            ]
        }
    ]
}
```

Every S3 case (read and write, any tags) returned `explicitDeny`. No statement in any attached policy matched, and the simulator reported `AllowedByOrganizations: false`. That points to an account-level restriction rather than a flaw in the policies, so all later tests evaluate the policy files directly with `simulate-custom-policy`.

An early attempt to pass the policy with `file://` failed with `Policy input list item 1 has invalid content`. The policy is passed as a compacted JSON string instead (`jq -c .`).

---

## Scenario 1: Standard new hire, Finance

Policy: `dept-readwrite.json`.

```bash
POLICY=$(jq -c . dept-readwrite.json)
sim() { aws iam simulate-custom-policy --policy-input-list "$POLICY" "$@" --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output text; }

echo "1. Read Finance/standard (expect allowed)"
sim --action-names s3:GetObject --resource-arns "arn:aws:s3:::example-bucket/*" --context-entries "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "2. Read HR/standard (expect implicitDeny)"
sim --action-names s3:GetObject --resource-arns "arn:aws:s3:::example-bucket/*" --context-entries "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=HR,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "3. Read Finance/restricted (expect implicitDeny)"
sim --action-names s3:GetObject --resource-arns "arn:aws:s3:::example-bucket/*" --context-entries "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=restricted,ContextKeyType=string"

echo "4. Write Finance/standard (expect allowed)"
sim --action-names s3:PutObject --resource-arns "arn:aws:s3:::example-bucket/*" --context-entries "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:RequestObjectTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:RequestObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "5. iam:CreateUser (expect implicitDeny)"
sim --action-names iam:CreateUser --resource-arns "*"
```

Output:

```
1. Read Finance/standard (expect allowed)
s3:GetObject    allowed
2. Read HR/standard (expect implicitDeny)
s3:GetObject    implicitDeny
3. Read Finance/restricted (expect implicitDeny)
s3:GetObject    implicitDeny
4. Write Finance/standard (expect allowed)
s3:PutObject    allowed
5. iam:CreateUser (expect implicitDeny)
iam:CreateUser  implicitDeny
```

Result: all five as expected.

---

## Scenario 2: HR hire with and without sensitive records access

Policies: `mfa-enforcement.json`, `dept-readwrite.json`, `hr-records.json`. Group membership is modeled by which policies are attached.

```bash
MFAPOL=$(jq -c . mfa-enforcement.json)
DEPT=$(jq -c . dept-readwrite.json)
HRREC=$(jq -c . hr-records.json)
S3="arn:aws:s3:::example-bucket/*"

tags() { printf '%s\n' "ContextKeyName=s3:$1/department,ContextKeyValues=$2,ContextKeyType=string" "ContextKeyName=s3:$1/sensitivity,ContextKeyValues=$3,ContextKeyType=string"; }
mapfile -t R_HR_RES < <(tags ExistingObjectTag HR restricted)
mapfile -t R_HR_STD < <(tags ExistingObjectTag HR standard)
mapfile -t R_FIN_RES < <(tags ExistingObjectTag Finance restricted)
mapfile -t W_HR_RES < <(tags RequestObjectTag HR restricted)

# usage: sim2 DEPT MFA ACTION [object tag entries...]   (policies come from the POLS array)
sim2() {
  local dept="$1" mfa="$2" action="$3"; shift 3
  aws iam simulate-custom-policy --policy-input-list "${POLS[@]}" \
    --action-names "$action" --resource-arns "$S3" \
    --context-entries \
      "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=$dept,ContextKeyType=string" \
      "ContextKeyName=aws:MultiFactorAuthPresent,ContextKeyValues=$mfa,ContextKeyType=boolean" \
      "$@" \
    --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output text
}

POLS=("$MFAPOL" "$DEPT" "$HRREC")
echo "1. nfoster (in records group): read HR/restricted (expect allowed)"
sim2 HR true s3:GetObject "${R_HR_RES[@]}"
echo "2. nfoster: write HR/restricted (expect allowed)"
sim2 HR true s3:PutObject "${W_HR_RES[@]}"
echo "5. nfoster: read Finance/restricted (expect implicitDeny)"
sim2 HR true s3:GetObject "${R_FIN_RES[@]}"
echo "6. nfoster, no MFA: read HR/restricted (expect explicitDeny)"
sim2 HR false s3:GetObject "${R_HR_RES[@]}"
echo "7. Finance employee wrongly given the HR records policy: read HR/restricted (expect implicitDeny)"
sim2 Finance true s3:GetObject "${R_HR_RES[@]}"

POLS=("$MFAPOL" "$DEPT")
echo "3. oadams (not in records group): read HR/restricted (expect implicitDeny)"
sim2 HR true s3:GetObject "${R_HR_RES[@]}"
echo "4. oadams: read HR/standard (expect allowed)"
sim2 HR true s3:GetObject "${R_HR_STD[@]}"
echo "8. oadams: write HR/restricted (expect implicitDeny)"
sim2 HR true s3:PutObject "${W_HR_RES[@]}"
```

Output:

```
1. nfoster (in records group): read HR/restricted (expect allowed)
s3:GetObject    allowed
2. nfoster: write HR/restricted (expect allowed)
s3:PutObject    allowed
5. nfoster: read Finance/restricted (expect implicitDeny)
s3:GetObject    implicitDeny
6. nfoster, no MFA: read HR/restricted (expect explicitDeny)
s3:GetObject    explicitDeny
7. Finance employee wrongly given the HR records policy: read HR/restricted (expect implicitDeny)
s3:GetObject    implicitDeny
3. oadams (not in records group): read HR/restricted (expect implicitDeny)
s3:GetObject    implicitDeny
4. oadams: read HR/standard (expect allowed)
s3:GetObject    allowed
8. oadams: write HR/restricted (expect implicitDeny)
s3:PutObject    implicitDeny
```

Result: all eight as expected.

---

## Scenario 3: New IT admin and self-escalation attempt

Policies: `mfa-enforcement.json`, `it-sysadmin.json`, `permission-boundary.json`, plus an over-broad `Allow *` test policy.

```bash
SYSADMIN=$(jq -c . it-sysadmin.json)
BASE=$(jq -c . mfa-enforcement.json)
BOUND=$(jq -c . permission-boundary.json)
BROAD='{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}'

sim()  { aws iam simulate-custom-policy --policy-input-list "$1" "${@:2}" --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output text; }
simb() { aws iam simulate-custom-policy --policy-input-list "$1" --permissions-boundary-policy-input-list "$BOUND" "${@:2}" --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output text; }

echo "A. sgarcia daily account (baseline only, MFA present)"
sim "$BASE" --action-names ec2:StartInstances --resource-arns "*" --context-entries "ContextKeyName=aws:MultiFactorAuthPresent,ContextKeyValues=true,ContextKeyType=boolean"

echo "B. sgarcia-admin (sysadmin policy + boundary)"
simb "$SYSADMIN" --action-names ec2:StartInstances iam:AttachUserPolicy iam:CreateAccessKey iam:PutUserPermissionsBoundary cloudtrail:StopLogging --resource-arns "*"

echo "C0. Over-broad policy, NO boundary (shows the risk)"
sim "$BROAD" --action-names iam:AttachUserPolicy cloudtrail:StopLogging --resource-arns "*"

echo "C1. Same over-broad policy, boundary attached"
simb "$BROAD" --action-names ec2:StartInstances iam:AttachUserPolicy iam:CreateAccessKey iam:PutUserPermissionsBoundary cloudtrail:StopLogging sso:CreateAccountAssignment --resource-arns "*"
```

Output:

```
A. sgarcia daily account (baseline only, MFA present)
ec2:StartInstances      implicitDeny
B. sgarcia-admin (sysadmin policy + boundary)
ec2:StartInstances      allowed
iam:AttachUserPolicy    explicitDeny
iam:CreateAccessKey     explicitDeny
iam:PutUserPermissionsBoundary  explicitDeny
cloudtrail:StopLogging  explicitDeny
C0. Over-broad policy, NO boundary (shows the risk)
iam:AttachUserPolicy    allowed
cloudtrail:StopLogging  allowed
C1. Same over-broad policy, boundary attached
ec2:StartInstances      allowed
iam:AttachUserPolicy    explicitDeny
iam:CreateAccessKey     explicitDeny
iam:PutUserPermissionsBoundary  explicitDeny
cloudtrail:StopLogging  explicitDeny
sso:CreateAccountAssignment     allowed
```

Result: all as expected. `sso:CreateAccountAssignment` being allowed is the known Identity Center limitation (risk R1).

---

## Scenario 4: Contractor before and after contract end

Policies: `contractor-limited.json` plus a per-contractor expiry policy.

```bash
LIMITED=$(jq -c . contractor-limited.json)
EXP_S=$(jq -c . contractor-expiry-sjenkins.json)
EXP_H=$(jq -c . contractor-expiry-hmorgan.json)
S3="arn:aws:s3:::example-bucket/*"

# usage: simc EXPIRY_POLICY DEPT TIME ACTION RESOURCE [extra context entries...]
simc() {
  aws iam simulate-custom-policy \
    --policy-input-list "$LIMITED" "$1" \
    --action-names "$4" --resource-arns "$5" \
    --context-entries \
      "ContextKeyName=aws:MultiFactorAuthPresent,ContextKeyValues=true,ContextKeyType=boolean" \
      "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=$2,ContextKeyType=string" \
      "ContextKeyName=aws:CurrentTime,ContextKeyValues=$3,ContextKeyType=date" \
      "${@:6}" \
    --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output text
}
```

### First run (before fixes)

```bash
echo "1. sjenkins, 2026-10-15, read Sales/standard (expect allowed)"
simc "$EXP_S" Sales 2026-10-15T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "2. sjenkins, 2026-10-15, write an object (expect explicitDeny)"
simc "$EXP_S" Sales 2026-10-15T12:00:00Z s3:PutObject "$S3"

echo "3. sjenkins, 2026-10-15, read Finance/standard (expect implicitDeny)"
simc "$EXP_S" Sales 2026-10-15T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Finance,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "4. sjenkins, 2026-10-15, iam:CreateUser (expect explicitDeny)"
simc "$EXP_S" Sales 2026-10-15T12:00:00Z iam:CreateUser "*"

echo "5. sjenkins, 2026-10-15, read Sales/RESTRICTED (design intent: deny)"
simc "$EXP_S" Sales 2026-10-15T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=restricted,ContextKeyType=string"

echo "6. sjenkins, 2027-01-05 (after contract end), read Sales/standard (expect explicitDeny)"
simc "$EXP_S" Sales 2027-01-05T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "7. hmorgan, 2026-12-01 (after his 2026-11-30 end), read Ops/standard (expect explicitDeny)"
simc "$EXP_H" Operations 2026-12-01T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Operations,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "8. sjenkins, 2026-12-01 (still under contract), read Sales/standard (expect allowed)"
simc "$EXP_S" Sales 2026-12-01T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"
```

Output:

```
1. sjenkins, 2026-10-15, read Sales/standard (expect allowed)
s3:GetObject    allowed
2. sjenkins, 2026-10-15, write an object (expect explicitDeny)
s3:PutObject    explicitDeny
3. sjenkins, 2026-10-15, read Finance/standard (expect implicitDeny)
s3:GetObject    implicitDeny
4. sjenkins, 2026-10-15, iam:CreateUser (expect explicitDeny)
iam:CreateUser  explicitDeny
5. sjenkins, 2026-10-15, read Sales/RESTRICTED (design intent: deny)
s3:GetObject    allowed
6. sjenkins, 2027-01-05 (after contract end), read Sales/standard (expect explicitDeny)
s3:GetObject    explicitDeny
7. hmorgan, 2026-12-01 (after his 2026-11-30 end), read Ops/standard (expect explicitDeny)
s3:GetObject    allowed
8. sjenkins, 2026-12-01 (still under contract), read Sales/standard (expect allowed)
s3:GetObject    allowed
```

Two failures: case 5 (contractor read restricted data, fixed in [change 2](04-policy-change-log.md)) and case 7 (`hmorgan` still allowed after his contract end because his expiry file contained `2026-12-31T23:59:59Z`, fixed in change 3).

### Date check after the fix

```bash
grep -E "sjenkins|hmorgan" ../../data/users-sample.csv | cut -d, -f1,9
grep -H "DateGreaterThan" contractor-expiry-*.json
```

```
sjenkins,2026-12-31
hmorgan,2026-11-30
contractor-expiry-hmorgan.json:        "DateGreaterThan": { "aws:CurrentTime": "2026-11-30T23:59:59Z" }
contractor-expiry-sjenkins.json:        "DateGreaterThan": { "aws:CurrentTime": "2026-12-31T23:59:59Z" }
```

### Retest after both fixes

```bash
LIMITED=$(jq -c . contractor-limited.json)
EXP_S=$(jq -c . contractor-expiry-sjenkins.json)
EXP_H=$(jq -c . contractor-expiry-hmorgan.json)

echo "R1. sjenkins, 2026-10-15, read Sales/standard (expect allowed)"
simc "$EXP_S" Sales 2026-10-15T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "R5. sjenkins, 2026-10-15, read Sales/restricted (expect implicitDeny)"
simc "$EXP_S" Sales 2026-10-15T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=restricted,ContextKeyType=string"

echo "R7. hmorgan, 2026-12-01, read Ops/standard (expect explicitDeny)"
simc "$EXP_H" Operations 2026-12-01T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Operations,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"

echo "R8. sjenkins, 2026-12-01, read Sales/standard (expect allowed)"
simc "$EXP_S" Sales 2026-12-01T12:00:00Z s3:GetObject "$S3" "ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string"
```

```
R1. sjenkins, 2026-10-15, read Sales/standard (expect allowed)
s3:GetObject    allowed
R5. sjenkins, 2026-10-15, read Sales/restricted (expect implicitDeny)
s3:GetObject    implicitDeny
R7. hmorgan, 2026-12-01, read Ops/standard (expect explicitDeny)
s3:GetObject    explicitDeny
R8. sjenkins, 2026-12-01, read Sales/standard (expect allowed)
s3:GetObject    allowed
```

Result: all four as expected after the fixes.

---

## Scenario 5: Department transfer (Sales to Finance)

Policies: `mfa-enforcement.json`, `dept-readwrite.json`, `hr-records.json`. The `department` tag is modeled as a context value.

```bash
MFAPOL=$(jq -c . mfa-enforcement.json)
DEPT=$(jq -c . dept-readwrite.json)
HRREC=$(jq -c . hr-records.json)
S3="arn:aws:s3:::example-bucket/*"

tags() { printf '%s\n' "ContextKeyName=s3:$1/department,ContextKeyValues=$2,ContextKeyType=string" "ContextKeyName=s3:$1/sensitivity,ContextKeyValues=$3,ContextKeyType=string"; }
mapfile -t R_SALES_STD < <(tags ExistingObjectTag Sales standard)
mapfile -t R_FIN_STD < <(tags ExistingObjectTag Finance standard)
mapfile -t R_HR_RES < <(tags ExistingObjectTag HR restricted)

sim2() {
  local dept="$1" mfa="$2" action="$3"; shift 3
  aws iam simulate-custom-policy --policy-input-list "${POLS[@]}" \
    --action-names "$action" --resource-arns "$S3" \
    --context-entries \
      "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=$dept,ContextKeyType=string" \
      "ContextKeyName=aws:MultiFactorAuthPresent,ContextKeyValues=$mfa,ContextKeyType=boolean" \
      "$@" \
    --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output text
}

POLS=("$MFAPOL" "$DEPT")

echo "A1. BEFORE transfer (tag Sales): read Sales/standard (expect allowed)"
sim2 Sales true s3:GetObject "${R_SALES_STD[@]}"
echo "A2. BEFORE transfer (tag Sales): read Finance/standard (expect implicitDeny)"
sim2 Sales true s3:GetObject "${R_FIN_STD[@]}"

echo "B1. Correct transfer (tag Finance): read Finance/standard (expect allowed)"
sim2 Finance true s3:GetObject "${R_FIN_STD[@]}"
echo "B2. Correct transfer (tag Finance): read Sales/standard (expect implicitDeny)"
sim2 Finance true s3:GetObject "${R_SALES_STD[@]}"

echo "C1. Partial: group changed, tag still Sales: read Finance/standard (new job access; expect implicitDeny)"
sim2 Sales true s3:GetObject "${R_FIN_STD[@]}"
echo "C2. Partial: group changed, tag still Sales: read Sales/standard (design intent: deny)"
sim2 Sales true s3:GetObject "${R_SALES_STD[@]}"

echo "D1. Partial: tag changed, old group not removed: read Sales/standard (expect implicitDeny)"
sim2 Finance true s3:GetObject "${R_SALES_STD[@]}"

POLS=("$MFAPOL" "$DEPT" "$HRREC")
echo "E1. HR employee transferred to Finance, tag updated, HR-records group NOT removed: read HR/restricted (expect implicitDeny)"
sim2 Finance true s3:GetObject "${R_HR_RES[@]}"
```

Output:

```
A1. BEFORE transfer (tag Sales): read Sales/standard (expect allowed)
s3:GetObject    allowed
A2. BEFORE transfer (tag Sales): read Finance/standard (expect implicitDeny)
s3:GetObject    implicitDeny
B1. Correct transfer (tag Finance): read Finance/standard (expect allowed)
s3:GetObject    allowed
B2. Correct transfer (tag Finance): read Sales/standard (expect implicitDeny)
s3:GetObject    implicitDeny
C1. Partial: group changed, tag still Sales: read Finance/standard (new job access; expect implicitDeny)
s3:GetObject    implicitDeny
C2. Partial: group changed, tag still Sales: read Sales/standard (design intent: deny)
s3:GetObject    allowed
D1. Partial: tag changed, old group not removed: read Sales/standard (expect implicitDeny)
s3:GetObject    implicitDeny
E1. HR employee transferred to Finance, tag updated, HR-records group NOT removed: read HR/restricted (expect implicitDeny)
s3:GetObject    implicitDeny
```

Finding: C2 returned `allowed`. Data access follows the `department` tag, so a transfer that changes groups but not the tag leaves old access. Fixed by reordering the transfer procedure (tag first), recorded as [change 4](04-policy-change-log.md). D1 and E1 show the tag-first order leaves no stale access.

---

## Scenario 6: User who has not enrolled MFA

Policies: `mfa-enforcement.json` + `dept-readwrite.json`.

```bash
MFAPOL=$(jq -c . mfa-enforcement.json)
DEPT=$(jq -c . dept-readwrite.json)
S3="arn:aws:s3:::example-bucket/*"
ACCT=123456789012

# usage: simm MFA(true|false|none) ACTION RESOURCE [extra context entries...]
simm() {
  local mfa="$1" action="$2" res="$3"; shift 3
  local ctx=("ContextKeyName=aws:username,ContextKeyValues=lbaker,ContextKeyType=string"
             "ContextKeyName=aws:PrincipalTag/department,ContextKeyValues=Sales,ContextKeyType=string")
  if [ "$mfa" != "none" ]; then
    ctx+=("ContextKeyName=aws:MultiFactorAuthPresent,ContextKeyValues=$mfa,ContextKeyType=boolean")
  fi
  aws iam simulate-custom-policy --policy-input-list "$MFAPOL" "$DEPT" \
    --action-names "$action" --resource-arns "$res" \
    --context-entries "${ctx[@]}" "$@" \
    --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output text
}
TAGS=("ContextKeyName=s3:ExistingObjectTag/department,ContextKeyValues=Sales,ContextKeyType=string" "ContextKeyName=s3:ExistingObjectTag/sensitivity,ContextKeyValues=standard,ContextKeyType=string")
```

### First run

```bash
echo "1. No MFA: create own virtual MFA device (expect allowed)"
simm false iam:CreateVirtualMFADevice "arn:aws:iam::$ACCT:mfa/lbaker"
echo "2. No MFA: enable own MFA device (expect allowed)"
simm false iam:EnableMFADevice "arn:aws:iam::$ACCT:user/lbaker"
echo "3. No MFA: read Sales/standard data (expect explicitDeny)"
simm false s3:GetObject "$S3" "${TAGS[@]}"
echo "4. No MFA: remove own MFA device (expect explicitDeny)"
simm false iam:DeactivateMFADevice "arn:aws:iam::$ACCT:mfa/lbaker"
echo "5. No MFA: iam:CreateUser (expect explicitDeny)"
simm false iam:CreateUser "*"
echo "6. MFA key absent entirely: read Sales/standard (expect explicitDeny)"
simm none s3:GetObject "$S3" "${TAGS[@]}"
echo "7. MFA present: read Sales/standard (expect allowed)"
simm true s3:GetObject
