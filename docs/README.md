# ShopEase documentation

Start here to find application guides, deployment-readiness research, and the original Online Boutique documentation.

## Find what you need

| I want to… | Read |
| --- | --- |
| Understand ShopEase, its users, ownership, and scope | [Product overview](product-overview.md) |
| Verify that a shopper can complete a purchase | [Critical user journey](critical-user-journey.md) |
| See the application dependencies and planned delivery path | [Product architecture](architecture.md) |
| Continue Stage 7 and understand the remaining actions | [Onboarding actions guide](<notes and research/onboarding-actions-guide.md>) |
| Run the application locally | [Local development](local-development/README.md) |
| Understand service calls and shopper request flows | [Interservice communication flows](<notes and research/interservice-communication-flows.md>) |
| Prepare the application for a platform handover | [Cloud handover guide](cloud-handover/README.md) |
| Review Stage 7 onboarding choices | [Onboarding research](<notes and research/onboarding-research.md>) |
| Revisit learning notes and platform questions | [Notes and research](<notes and research/README.md>) |

## How the documentation is organized

```text
docs/
  README.md                  Start here
  product-overview.md        ShopEase purpose, users, ownership, and scope
  critical-user-journey.md   Repeatable purchase steps and pass/fail checks
  architecture.md           Application and planned delivery diagrams
  local-development/         Running services, troubleshooting, and session evidence
  cloud-handover/            Readiness checklist and 18 detailed research chapters
  notes and research/        Onboarding decisions, service flows, and platform notes
  img/                       Existing upstream images
  releasing/                 Existing upstream release documentation and scripts
  Other root guides          Existing upstream reference guides listed below
```

The handover chapters are research and checklists, not proof that deployment work is complete. The local session log records historical observations. Use the indexes to distinguish practical guides, proposals, and evidence.

Application source, Dockerfiles, tests, CI, and this documentation belong in `akansof-org/shopease`. PlatformOne deployment desired state belongs in `platformone-gitops/apps/shopease/`; Argo CD registration belongs in that repository's `applications/` and `appprojects/` directories.

## Upstream reference documentation

These documents retain their existing locations to preserve upstream links and tooling. They describe Online Boutique and its Google Cloud workflows; they are not the ShopEase PlatformOne deployment procedure.

| Document | Purpose |
| --- | --- |
| [Development guide](development-guide.md) | Upstream Skaffold and Kubernetes development workflow |
| [Adding a microservice](adding-new-microservice.md) | Upstream service contribution conventions |
| [Purpose](purpose.md) | Upstream project's goals |
| [Product requirements](product-requirements.md) | Upstream requirements for contributions and demos |
| [Cloud Shell tutorial](cloudshell-tutorial.md) | Upstream interactive Kubernetes quickstart |
| [DeployStack](deploystack.md) | Upstream Google Cloud deployment entry point |
| [Releasing](releasing/README.md) | Upstream release process and supporting scripts |

## Adding documentation

Put repeatable local instructions in `local-development/`, handover assessment material in `cloud-handover/`, and exploratory findings or draft decisions in `notes and research/`. Add a link in the relevant index. Label dated evidence and proposals explicitly, and record test results before marking readiness checks complete.

Unless a document says otherwise, run application commands from the repository root. Some research links point to sibling PlatformOne repositories in the local Akansof workspace and require that workspace layout.
