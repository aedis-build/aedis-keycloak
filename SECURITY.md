# Security policy

## Supported versions

Only the latest release. Every release tracks the latest Keycloak patch of the current Keycloak minor; a new Keycloak
minor becomes the supported line within about 30 days of its release, because the previous minor stops receiving
upstream fixes. Pin the image and the chart by digest, never by a moving tag.

## Reporting a vulnerability

Use **Report a vulnerability** in this repository's Security tab (GitHub Private Vulnerability Reporting). Do not open a
public issue for vulnerabilities.

- Acknowledgement within 3 business days.
- Initial severity assessment (CVSS) within 7 days.
- Fix or documented mitigation for high/critical severity within 30 days; a high/critical Keycloak advisory is shipped
  in a new release within 72 hours of the upstream patch.
- Credit to the reporter in the advisory, if desired.

## Scope

What this repository adds on top of Keycloak: the image build, the SPIs, the Helm chart, the baseline realm and the
tests. Vulnerabilities in Keycloak itself go to [keycloak/keycloak](https://github.com/keycloak/keycloak/security/policy);
in the Aedis Platform reusable workflows this repository calls, to
[aedis-build/aedis-platform](https://github.com/aedis-build/aedis-platform/security/policy).

## Practices of this repository

- Every action and image pinned by commit SHA or digest; `permissions: {}` by default.
- Actions updated by Dependabot (and the Keycloak image by Renovate, once the image lands); OpenSSF Scorecard published.
- This is not an official Keycloak distribution and is not endorsed by the Keycloak project.
