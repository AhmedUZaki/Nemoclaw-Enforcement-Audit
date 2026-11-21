# Environment versions

Recorded on 2026-11-21, from the machine the tests run on.

| Component | Version |
|---|---|
| NemoClaw | v0.0.140 (git tag; the CLI reports `v0.1.0`), commit `88fe14f3fb9a0bf72997cce19ec88b5b2ee08391` |
| OpenShell | 0.0.116 |
| Node.js | v22.23.5 (installed by NemoClaw's installer, through nvm) |
| Docker | 29.9.3 (client and server) |
| Ubuntu (WSL distro) | 24.04.5 LTS |
| WSL | 3.0.5.0 |
| Linux kernel (WSL 2) | 6.18.58.1-microsoft-standard-WSL2 |
| Windows | 11 Education, 10.0.26200 |

Documentation version: the OpenShell claims in `CLAIMS.md` come from the "latest" docs page, which shows v0.1.9. The installed OpenShell is 0.0.116, so a claim may describe behavior this version does not have.

Agent: none onboarded. NemoClaw was installed with `--defer-onboarding`, so there is no sandbox, no model provider and no agent model yet.

GPU: NVIDIA GeForce RTX 3080, 10,240 MiB VRAM. Not used by the current setup (no local model is installed).
