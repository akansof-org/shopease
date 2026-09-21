# Upstream attribution

ShopEase is an Akansof-maintained fork of **Online Boutique**, originally developed by **Google and the GoogleCloudPlatform/microservices-demo contributors**. The original application, service design, and inherited documentation are their work. Akansof's changes adapt the project for local development and PlatformOne onboarding.

## Source and baseline

| Item | Value |
| --- | --- |
| Original project | [Google Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) |
| ShopEase fork | [akansof-org/shopease](https://github.com/akansof-org/shopease) |
| Actual upstream starting commit | [`9a4616e77f0f9cbcbecaf27d711c38890dda1404`](https://github.com/GoogleCloudPlatform/microservices-demo/commit/9a4616e77f0f9cbcbecaf27d711c38890dda1404) |
| Starting commit subject | `release/v0.10.6 (#3432)` |
| Related upstream release | [`v0.10.6`](https://github.com/GoogleCloudPlatform/microservices-demo/releases/tag/v0.10.6) |
| Release tag commit | `5b3a712ab85ccb8f6f7cd5b720d36ba9a8d041eb` |
| Inherited licence | [Apache License 2.0](LICENSE) |

The starting commit is the parent of the first ShopEase customization commit, `85baeed3`. It is **not** the commit referenced by the `v0.10.6` tag. Earlier research proposed starting from that tag; the existing local work was instead preserved on its original upstream starting commit. This record describes the actual history, without claiming that the checkout was reset to the release tag or that release validation is complete.

## Akansof modifications

The initial ShopEase changes include:

- Service Dockerfile adjustments for local builds, including architecture arguments and frontend build settings.
- Executable permission for the ad service Gradle wrapper.
- Docker Compose configuration for local execution.
- Local development guides, troubleshooting evidence, cloud handover research, and onboarding documentation.
- Documentation organization and ignore rules for generated build outputs.

Git history records the exact changes. To compare committed ShopEase work against the original baseline, run from the repository root:

```bash
git diff 9a4616e77f0f9cbcbecaf27d711c38890dda1404 HEAD
```

PlatformOne deployment configuration is maintained separately in `akansof-org/platformone-gitops`, under `apps/shopease/`. Its folder scaffolding is not evidence of a completed deployment. The inherited GitHub workflows and release scripts describe upstream automation until explicitly adapted and verified for ShopEase.

## Licence and credit

The upstream `LICENSE` and inherited copyright headers are retained. ShopEase does not claim authorship of the original application. Keep applicable upstream and third-party notices when modifying or redistributing their material, and identify ShopEase changes in the affected files and release history.

No root `NOTICE` file exists in the recorded upstream baseline. Preserve any applicable notices provided by future upstream updates or dependencies. This attribution document supplements the inherited licence and notices; it does not replace them or imply Google endorsement of ShopEase.

## Reviewing upstream updates

The local remote convention is:

```text
origin   → https://github.com/akansof-org/shopease.git
upstream → https://github.com/GoogleCloudPlatform/microservices-demo.git
```

Fetch upstream changes, review the proposed release or commits on a dedicated branch, and integrate them through a pull request into the ShopEase fork. Check compatibility with local modifications, run relevant tests and image builds/scans, and update this baseline record when an upstream update is accepted. Do not automatically overwrite ShopEase changes with upstream state.
