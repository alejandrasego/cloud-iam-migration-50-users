# Write-up: Cloud IAM Migration (50 Accounts)

## Failure cases
(Fill in after testing. Link each to a scenario in `03-test-scenarios.md` and a change in `04-policy-change-log.md`.)

## Remaining security limitations
- MFA is enforced at the Identity Center layer; the IAM policy is a backstop for non-federated paths.
- The permission boundary does not cover Identity Center actions (`sso-admin:*`); an identity admin could self-assign broad access. Mitigated by SCP and alerting (not yet implemented).
- The boundary is a deny-list on top of `Allow *`; any new risky AWS action is not blocked until added.
- The help desk policy uses IAM resource tags to separate privileged users. In Identity Center, an equivalent restriction needs a different mechanism.
- Contractor expiry uses a hardcoded date per contractor, which does not scale, and it does not end sessions already issued.
- S3 cannot filter `ListBucket` by object tags, so department users can see object names across buckets.
- The test account is on the AWS free plan, where IAM user groups are unavailable. Test policies were attached directly to test users instead of through groups.
- Lab work was performed from the account owner's sign-in. A production deployment would use a separate named admin identity.

## Next improvements
(Fill in at the end. Candidates: SCP for Identity Center assignments, SCIM provisioning from an HR system, quarterly access recertification, SIEM alerting on IAM changes, allow-list boundary design.)
