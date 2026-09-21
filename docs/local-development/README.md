# Local development

[Documentation home](../README.md)

| Document | Use it for | Type |
| --- | --- | --- |
| [Docker Compose](local-docker-compose.md) | Starting the application as containers and optionally generating load | Practical guide |
| [Local services guide](local-services-guide.md) | Learning dependencies and running individual services directly | Practical guide |
| [Local issues and fixes](local-run-issues-and-fixes.md) | Understanding problems encountered during local execution and the fixes used | Troubleshooting notes, dated 2026-08-04 |
| [Local run session](local-run-test.md) | Inspecting original commands, outputs, and observations | Historical session evidence |

Start with Compose to run the application, or the services guide to study it one service at a time. Use the issues-and-fixes document when debugging; consult the raw session only when you need the original evidence.

Run commands from the repository root unless instructed otherwise. The session log retains the old `10-downloads/shopease` paths as historical output; the checkout now lives at `04-products/shopease`.

For dependencies and request paths, see [interservice communication flows](<../notes and research/interservice-communication-flows.md>). For deployment preparation, continue to [cloud handover](../cloud-handover/README.md).
