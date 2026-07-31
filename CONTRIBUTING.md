# Contributing

Thanks for your interest in the collection. Bug reports, feature ideas, and PRs are all welcome.

## Quick start

The collection must live at the canonical path for FQCN resolution:

```bash
mkdir -p ~/dev/collections/ansible_collections/maksimrudakov
git clone https://github.com/MaksimRudakov/ansible-collection-rke2.git \
  ~/dev/collections/ansible_collections/maksimrudakov/rke2
cd ~/dev/collections/ansible_collections/maksimrudakov/rke2
export ANSIBLE_COLLECTIONS_PATH=~/dev/collections
```

## Testing

```bash
pip install "molecule>=25.1" "molecule-plugins[docker]>=23.7" "ansible-core>=2.16" ansible-lint yamllint
ansible-galaxy collection install ansible.posix community.docker

yamllint .
ansible-lint                # production profile
molecule test -s airgap     # fast: mock mirror, URL rewrites, downgrade guard
molecule test               # full: real RKE2 bootstrap in a privileged container
```

The `default` scenario runs real RKE2 in docker and needs the quirks already encoded in the scenario: `/var/lib/rancher` volume, `mount --make-rshared /`, curl in prepare. Don't remove them.

What has been verified end-to-end on real clusters (and on which versions) is tracked in [TESTING.md](./TESTING.md).

## Code style

- 2 spaces, YAML without redundant quotes, FQCN module names.
- Idempotency by default: re-running any playbook must be a no-op on a converged cluster.
- Variables use the `rke2_` prefix; every new variable goes into `roles/node/meta/argument_specs.yml` and README.
- `no_log` on tasks touching the token or registry credentials.
- Orchestration (serial, delegation order) belongs in `playbooks/`; node-level convergence in the role.

## Commit style

[Conventional Commits](https://www.conventionalcommits.org/) — `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`.

## Pull requests

- Branch off `main`, one logical change per PR.
- Update `CHANGELOG.md` under `## [Unreleased]`.
- CI must be green before review.

## Releasing (maintainers)

1. Bump `version` in `galaxy.yml`.
2. Move `## [Unreleased]` section in `CHANGELOG.md` to a new `## [X.Y.Z]` section.
3. Tag: `git tag -s vX.Y.Z -m "vX.Y.Z" && git push --tags`.
4. The release workflow verifies the version, builds the tarball, creates the GitHub release and publishes to Galaxy when `GALAXY_API_KEY` is configured.

## Reporting security issues

See [SECURITY.md](./SECURITY.md). Do **not** open public issues for vulnerabilities.
