# cloud-iam-migration-50-users
Migration plan, configuration, and testing for moving 50 local user accounts to cloud IAM. Includes 10 onboarding test scenarios, privilege escalation risk register, and a documented policy change. Aligned to NIST 800-53 and CIS Controls v8.
## Project Status

**Current phase: testing in progress**

### Completed
- [x] Sample data for 50 local user accounts (`data/`)
- [x] Initial cloud IAM configuration (`config/`)
- [x] Mapping of security controls to NIST 800-53 and CIS Controls v8

### In Progress
- [ ] Testing the configuration against 10 distinct user onboarding scenarios

### Planned
- [ ] Document one policy change made after catching an access control failure in testing
- [ ] Privilege escalation risk evaluation and mitigation notes
- [ ] Short write-up: failure cases, remaining security limitations, and next improvements

*This README is updated as each item is completed.*
