## Summary

<!-- What does this PR do? 1-3 bullet points. -->

-

## Test Plan

<!-- How was this tested? -->

- [ ] `ansible-playbook playbooks/<playbook>.yml --syntax-check`
- [ ] `ansible-lint playbooks/ remediation/ roles/`
- [ ] Ran with `--check` against test host
- [ ] Applied to real host and verified

## Checklist

- [ ] No hardcoded IPs, hostnames, or secrets (`grep -r "192.168\|10.2" .` returns nothing)
- [ ] Task names follow `"SECTION | Description"` convention
- [ ] All new tasks have appropriate tags (`deploy`, `configure`, `validate`)
- [ ] Destructive tasks guarded with `when: not ansible_check_mode`
- [ ] Updated README if adding a new playbook or role
