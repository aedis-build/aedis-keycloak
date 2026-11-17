# Design

Status: **proposal**, 2026-11-17. Nothing below is built yet; this document is the target the first pull requests
implement. It is not an official Keycloak distribution and is not endorsed by the Keycloak project.

## Goal

A Keycloak that is secure by construction and stays that way:

- a hardened, optimized image with the Aedis SPIs;
- a Helm chart whose defaults are the safe choice, so an unsafe setting has to be asked for explicitly;
- a hardened baseline realm with FAPI 2.0 on by default;
- SAST and DAST running against the installed model on every change;
- the latest Keycloak patch, applied automatically when every check passes.

## Layers

```text
quay.io/keycloak/keycloak:<version>@sha256   upstream
  └─ ghcr.io/aedis-build/keycloak:<version>  this repository: optimized build, minimal features, SPIs, base theme, signed
```

The image is built ahead of time (`kc.sh build`) because a read-only root filesystem needs an optimized image: the
upstream image builds on start and writes to its own directory. Brand-specific theming is configuration, not a new
image.

## Repository layout

| Path | Contents |
| --- | --- |
| `image/` | Dockerfile from the upstream image by digest; `kc.sh build` with the minimal feature list (no `scripts`, no `device-flow`); cosign signature, SBOM and provenance |
| `spi/` | Java SPIs: namespaced users in a shared realm, an event listener that exports login and admin events, and the delegation mapper if the platform needs one |
| `charts/aedis-keycloak/` | Helm chart with safe defaults; `values.schema.json` rejects unsafe combinations unless an explicit `allowInsecure…` key is set |
| `realms/baseline/` | the hardened baseline realm, applied as code |
| `tests/` | outside-in probes, cross-organization test, FAPI 2.0 conformance plan |
| `ci/` | values per cloud (`aws`, `azure`, `generic`) |

## Chart defaults

- Public port 8080 separate from the management port 9000; health and metrics are never exposed.
- The admin console on an internal hostname only.
- Ingress limited to the minimal list of public paths.
- `proxy-headers` with explicit trusted proxies; the forwarded client address is trusted only from them.
- Non-root, read-only root filesystem (`emptyDir` where Keycloak writes), seccomp `RuntimeDefault`, all capabilities
  dropped, no service account token mounted.
- NetworkPolicy denying everything by default and allowing only the database.
- PodDisruptionBudget, External Secrets, image by digest.
- The realm applied as code by a job (keycloak-config-cli), which also allows comparing the live realm with the
  repository.
- No operator: the chart runs on any cluster and in kind. Whether an upgrade can roll or must recreate is decided by a
  pre-upgrade hook running `kc.sh update-compatibility check`.

## Baseline realm

- Direct access grants off in every client, including the built-in ones (`admin-cli` too); device flow, dynamic
  client registration and self-registration off.
- Brute-force detection with temporary lockout and growing wait times, plus per-IP rate limits at the edge (the
  built-in detection is per user, not per IP).
- Refresh token rotation on; default group with no business role.
- Login and admin events on and exported to durable storage.
- FAPI 2.0 client policies (`fapi-2-dpop-security-profile`) applied by default.
- The `master` realm treated as production: password policy, brute-force detection, small admin group, admin console
  off the internet.

## Tests and CI

- **Static:** analysis of the templates and the rendered manifests (Trivy misconfiguration via
  [aedis-sca](https://github.com/aedis-build/aedis-sca), `helm lint`, chart-testing, kubeconform,
  [aedis-conformance](https://github.com/aedis-build/aedis-conformance) level 2) and CodeQL on the SPIs.
- **Dynamic:** install the chart in kind with the baseline realm and run [aedis-dast](https://github.com/aedis-build/aedis-dast)
  (ZAP, Nuclei, edge checks), the outside-in probes (direct grant, device flow, dynamic registration and
  self-registration must all fail), the cross-organization test and the OpenID Foundation FAPI 2.0 plan.
- **Release:** image and chart (OCI on GHCR) signed with cosign.

## Updates

Keycloak ships a patch every two to three weeks, and a minor stops receiving fixes almost as soon as the next minor
is out. Two tracks:

1. **Patch, automatic.** Renovate opens a pull request that changes only the image tag and digest. With every check
   green, it merges and a new image and chart are released.
2. **Minor, quarterly.** Renovate opens the pull request; it never merges by itself. Read the release diff and the
   behaviour changes, rebuild and test the SPIs, change the image in one step and the features in another (never
   revert both together), and move to the new minor within about 30 days.

Renovate pull requests only touch the image. The feature list and the realms are owned by `CODEOWNERS` and never
merge automatically. Keep the SPIs few: each one can break on a minor upgrade.
