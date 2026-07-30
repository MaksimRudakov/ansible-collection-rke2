## Summary

<!-- What does this PR do and why? Link to the issue if applicable. -->

## Type of change

- [ ] Bug fix
- [ ] New feature
- [ ] Refactor
- [ ] Docs / examples
- [ ] CI / build

## Checklist

- [ ] `yamllint .` passes
- [ ] `ansible-lint` passes (production profile)
- [ ] `molecule test -s airgap` passes
- [ ] `molecule test` passes for changes affecting install/bootstrap
- [ ] `CHANGELOG.md` updated under `## [Unreleased]`
- [ ] `meta/argument_specs.yml` updated for new/changed variables
- [ ] No secrets, tokens, or real hostnames/IPs in diff
