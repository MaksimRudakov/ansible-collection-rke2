# Security Policy

## Supported versions

Only the latest minor release receives security fixes.

## Reporting a vulnerability

**Please do not report security vulnerabilities through public GitHub issues, discussions, or pull requests.**

Use GitHub's private vulnerability reporting:

1. Go to https://github.com/MaksimRudakov/ansible-collection-rke2/security/advisories/new
2. Provide a clear description, reproduction steps, and impact assessment.
3. We will acknowledge within 72 hours.

## Scope

In scope:
- Handling of the cluster join token and registry credentials (file modes, `no_log`, delegation).
- The air-gapped install path (install script download and URL rewriting).
- Rendered configuration (`config.yaml`, `registries.yaml`, Cilium `HelmChartConfig`).

Out of scope:
- Vulnerabilities in RKE2, Cilium, or Ansible themselves — report those upstream.
- Misconfiguration on the operator's side (world-readable kubeconfig by choice, plaintext secrets in inventory instead of ansible-vault, etc.).

## Disclosure

We follow [coordinated disclosure](https://en.wikipedia.org/wiki/Coordinated_vulnerability_disclosure). After a fix is released, we publish a GitHub Security Advisory with credit to the reporter (unless anonymity is requested).
