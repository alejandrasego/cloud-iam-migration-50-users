# Write-up: Cloud IAM Migration (50 Accounts)

## Failure cases
**1. Permission boundary blocked the help desk (scenario 8, policy change 1).** Two policies that each passed testing on their own conflicted when combined: the boundary's blanket deny on password resets blocked the help desk's legitimate resets. Found by testing the help desk policy with and without the boundary, fixed with a conditional deny, and retested (see [`04-policy-change-log.md`](04-policy-change-log.md)). Lesson: test policies in combination, not only individually.

**2. Account-level guardrail masked the policy results (testing environment).** Initial tests against the live test account returned `explicitDeny` for every S3 action regardless of tags. The simulator showed no matching statement in any attached policy and `AllowedByOrganizations: false`, which pointed to an account-level guardrail rather than a flaw in the policy. Testing was redone with `simulate-custom-policy`, which evaluates the policy logic in isolation.

**3. Two inconsistencies in contractor access (scenario 4, policy changes 2 and 3).** The contractor policy still allowed reads of restricted data after the same rule was added to the employee policy, and one contractor's expiry file contained another contractor's end date. Both were found by testing and fixed. Lesson: when a rule changes in one policy, check every policy that should follow it, and don't hand-copy per-person values.

**4. Transfer procedure left old access in place (scenario 5, policy change 4).** Data access in the design follows the `department` tag, but the transfer procedure put group changes first. A transfer that changed groups but not the tag left the user with their old department's data. Found by testing partial transfers, fixed by making the tag the first step. Lesson: identify which attribute actually controls access, and put it first in any procedure that changes it.

## Remaining security limitations
- MFA is enforced at the Identity Center layer; the IAM policy is a backstop for non-federated paths.
- The permission boundary does not cover Identity Center actions (`sso-admin:*`); an identity admin could self-assign broad access. Mitigated by SCP and alerting (not yet implemented).
- The boundary is a deny-list on top of `Allow *`; any new risky AWS action is not blocked until added.
- The help desk policy uses IAM resource tags to separate privileged users. In Identity Center, an equivalent restriction needs a different mechanism.
- Contractor expiry uses a hardcoded date per contractor, which does not scale, and it does not end sessions already issued.
- S3 cannot filter `ListBucket` by object tags, so department users can see object names across buckets.
- The test account is on the AWS free plan, where IAM user groups are unavailable. Test policies were attached directly to test users instead of through groups.
- Lab work was performed from the account owner's sign-in. A production deployment would use a separate named admin identity.
- Testing used policy-level simulation (`simulate-custom-policy`) with simulated tag values, not live users and resources, because account-level guardrails blocked S3 and the free plan restricted other features. Results verify policy logic, not end-to-end behavior.
-  Session revocation (`revoke-sessions.json`) covers temporary credentials only. Long-term IAM access keys must be deactivated separately at termination.
- The users of the shared front desk account are not identified in the inventory, so named replacement accounts cannot yet be created for them.
- Scenarios 9 and 10 are walkthroughs and inventory checks. They verify the plan and the policy logic, not live account disabling, deletion, or log attribution.
- The backup role's trust policy was reviewed by inspection, not simulated, because the policy simulator evaluates identity permissions only. That no human can assume the role rests on the trust policy naming only the backup service plus a human's own policy granting no assume-role.

## Next improvements
(Fill in at the end. Candidates: SCP for Identity Center assignments, SCIM provisioning from an HR system, quarterly access recertification, SIEM alerting on IAM changes, allow-list boundary design.)
- Generate contractor expiry policies from the inventory (`contract_end`) and add an automated check that dates match, so a copy-paste error like policy change 3 can't happen.
- Share the sensitivity rule across policies (or test every permission set against it) so policies can't drift apart, as in policy change 2.
- Derive the department from the identity source (directory or HR record) as an Identity Center attribute, so one record drives both tags and groups and a transfer cannot be half-finished.
- Add a periodic report comparing each user's `department` tag to their groups to catch missed transfer steps.
