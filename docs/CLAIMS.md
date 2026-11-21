# Claims to test

Every statement below is quoted or closely paraphrased from one of the official sources listed in References, with an inline citation. No interpretation, comparison, or question has been added beyond what each source states. Short quotes are exact; everything else is a paraphrase of the cited page.

The claims come from the "latest" documentation (OpenShell v0.1.9 at the time of writing). The installed OpenShell is 0.0.116 (see `VERSIONS.md`), so a claim can describe behavior this version does not have.

There are two phases:

- **Phase 1: OpenShell on its own.** Controlled probes against the sandbox engine with policies written for the test, no agent involved.
- **Phase 2: NemoClaw and the agent (deferred).** The policies NemoClaw layers on top of OpenShell. The agent will exercise these later, in real use, once it is onboarded.

---

# Phase 1: OpenShell on its own

## Enforcement layers

| ID | Layer | Mechanism | When it can change |
|---|---|---|---|
| E1 | Network | CONNECT proxy + OPA policy engine | At runtime (`openshell policy update` / `openshell policy set`) |
| E2 | Filesystem | Landlock LSM, kernel level | Only by re-creating the sandbox |
| E3 | Process | Seccomp BPF + non-root identity without Linux capabilities | Only by re-creating the sandbox |
| E4 | Provider credentials | Network policy + profile endpoint binding + proxy substitution | At runtime |

[4]

## Network

- **N1, deny-by-default egress.** All egress is denied; only endpoints listed in `network_policies` can receive traffic. [4]
- **N2, network namespace isolation.** Always active: "The sandbox cannot bypass the proxy at the network level." Docker and Podman disable networking on the workload container; Kubernetes uses NetworkPolicy; VM guests have no network device and use vsock. [4]
- **N3, binary identity binding.** Every `network_policies` entry requires a `binaries` list (`*` matches one path component, `**` recursive). The first SHA256 digest of a binary is pinned (trust on first use), and "a later digest mismatch triggers an immediate deny." [4]
- **N4, L4 vs L7.** An endpoint without `protocol` applies no request rules: the proxy still terminates TLS and parses HTTP, but "allows any HTTP method and path." `protocol: tcp` also applies no request rules. `protocol: rest` with `rules` restricts methods and paths; `websocket` and `graphql` also exist. [4]
- **N5, enforcement mode.** Default is `audit`: "The proxy logs violations but forwards traffic." Setting `enforce` blocks non-matching requests with a 403 response. [4]
- **N6, TLS handling.** Default is auto-detect and terminate, using a per-sandbox ephemeral CA injected into the sandbox trust stores. `tls: skip` on an endpoint disables placeholder credential rewriting, dynamic token grant injection and L7 inspection. [4]
- **N7, SSRF protection.** After the policy allows a connection, DNS is resolved and private addresses are blocked for undeclared, wildcard and hostless endpoints. Exact hostnames may resolve to RFC 1918 addresses. "Loopback (`127.0.0.0/8`), link-local (`169.254.0.0/16`), and unspecified (`0.0.0.0`) addresses are always blocked and cannot be overridden with `allowed_ips`." [4]
- **N8, operator approval.** Enabled by default: the proxy blocks unlisted endpoints and requires approval. An approved endpoint persists within the same sandbox instance and resets when the sandbox is destroyed and re-created. [4]

## Filesystem (Landlock)

- **F1, mandatory baseline.** The baseline "requires Landlock ABI 3 or newer, introduced in Linux 6.2," and startup fails if it cannot be enforced. [4]
- **F2, additional filesystem policy.** Defaults to `compatibility: best_effort`, which "may skip unavailable policy paths" and emits a High-severity OCSF `DetectionFinding` when the policy cannot be fully applied. `compatibility: hard_requirement` rejects startup instead. Neither setting relaxes the private-file baseline. [4]
- **F3, read-only and read-write paths.** System paths (`/usr`, `/lib`, `/etc`, `/var/log`) are read-only; the working directory and `/tmp` are read-write. Risk if system paths are made writable: the agent can replace binaries, modify TLS trust stores or change DNS resolution. [4]
- **F4, path validation.** Policy paths must be absolute, must not contain `..`, read-write paths must not be overly broad (such as `/`), each path is at most 4096 characters, and `read_only` plus `read_write` together are at most 256. Violations are rejected with `INVALID_ARGUMENT`. [4]

## Process

- **P1, privilege drop.** The runtime starts as a non-root user with no Linux capabilities and `no_new_privs` set; before exec it sets `RLIMIT_CORE=0` and `PR_SET_DUMPABLE=0`. `run_as_user` / `run_as_group` accept `sandbox` or a numeric ID from 1 to 4294967294; root and the invalid-identity sentinel are always rejected. [4]
- **P2, seccomp filters.** A startup prelude blocks `mount`, the new mount API syscalls, `pivot_root`, `umount2`, `bpf`, `perf_event_open`, `userfaultfd`, module loading and kexec. Socket families other than `AF_UNIX`, `AF_NETLINK`, `AF_INET` and `AF_INET6` fail with `EPERM`; among netlink protocols only `NETLINK_ROUTE` is allowed. The runtime also blocks `memfd_create`, `ptrace`, `process_vm_readv`/`writev`, `pidfd_*`, `io_uring_setup`, `setns` and others, plus conditional denials for `execveat` with `AT_EMPTY_PATH`, `unshare`/`clone` with `CLONE_NEWUSER`, and `seccomp(SECCOMP_SET_MODE_FILTER)`. "This is not a user-facing knob." [4]

## Credentials and inference

- **I1, no provider access by default.** Default is "No model-provider access." A provider must be attached, and policy must admit the native endpoint and the calling binary. [4]
- **I2, credential placeholder.** The agent gets "an opaque placeholder, not the real API key or access token"; the placeholder resolves only for the hosts, ports and paths the provider profile authorizes. [4]

---

# Phase 2: NemoClaw and the agent (deferred)

These are NemoClaw's own claims on top of OpenShell. They are kept here for when the agent is onboarded and exercises them in real use.

## Network baseline (applied regardless of tier)

| Group | Host(s) : port | Allowed |
|---|---|---|
| `nvidia` | integrate.api.nvidia.com:443 | POST to inference/embedding paths, GET to model listings |
| `clawhub` | clawhub.ai:443 | GET, POST, including scoped package paths |
| `openclaw_api` | openclaw.ai:443, catalog.openclaw.ai:443 | GET+POST on openclaw.ai; GET only on catalog.openclaw.ai |
| `openclaw_docs` | docs.openclaw.ai:443 | GET only |
| `npm_registry` | registry.npmjs.org:443 | GET only |

[1]

"GitHub access (`github.com`, `api.github.com`) is not included in the baseline policy." Apply the `github` preset during onboarding if needed. [1]

## Filesystem baseline (applied regardless of tier)

- Read-write: `/sandbox`, `/tmp`, `/dev/null`, `/dev/pts` [1][2]
- Read-only: `/usr`, `/lib`, `/proc`, `/dev/urandom`, `/app`, `/etc`, `/var/log`, `/var/lib/dpkg` [1][2]

"NemoClaw uses OpenShell filesystem policy to restrict access outside the writable agent state tree." [2]

"The container mounts system directories read-only to prevent the agent from modifying binaries, libraries, or configuration files." Risk if relaxed: "Making `/usr` or `/lib` writable lets the agent replace system binaries (such as `curl` or `node`) with trojanized versions." Recommendation: "Never make system paths writable. If the agent needs a writable location for generated files, use a subdirectory of `/sandbox`." [2]

Recommendation for writable paths: "Keep writable paths to `/sandbox` and `/tmp`. If the agent needs a persistent working directory, create a subdirectory under `/sandbox`." [2]

## Tiers (network presets added on top of the baseline)

| Tier | Presets added |
|---|---|
| Restricted | "No tier defaults" |
| Balanced (default) | `npm`, `pypi`, `huggingface`, `brew-balanced`, `brave`; selected `tavily` web search access when supported |
| Open | `npm`, `pypi`, `huggingface`, `brew`, `brave`, `weather`, `public-reference`, `slack`, `discord`, `telegram`, `wechat` (experimental), `whatsapp` (experimental), `teams` (experimental), `jira`, `outlook`; selected `tavily` web search access when supported |
| Personal | `personal-open-internet` (mandatory) |

[1]

"OpenClaw onboarding also adds the `openclaw-pricing` preset on top of Balanced and Open tier defaults." It is suppressed on Personal ("Personal suppresses the overlapping pricing route") and on Restricted ("Restricted suppresses it to retain the restricted posture"). [1]

"OpenClaw can select `brave` or `tavily`, while Hermes can select `tavily` only." On fresh onboarding, "OpenClaw suggests `brave` only when you select Brave Search" (similarly for Tavily). For the Personal tier: "`web_search` still requires a separately configured provider." [1]

## `/sandbox/.openclaw`

"The `/sandbox/.openclaw` directory contains the OpenClaw gateway configuration (model routing, CORS settings, channel config)." "This directory remains writable so the agent can manage its own config, install skills, and write standard home-directory state." Risk: "A writable `.openclaw` directory lets the agent modify its own gateway config: disabling CORS or redirecting inference to an attacker-controlled endpoint." [2]

## Landlock enforcement per agent

**OpenClaw** [2]: Default: "`compatibility: best_effort`. The entrypoint applies Landlock rules when the kernel supports them and silently skips them on older kernels." Risk: "On kernels without Landlock support (pre-5.13), filesystem restrictions rely solely on container mount configuration, which is less granular." Recommendation: "Run on a kernel that supports Landlock (5.13+). Ubuntu 22.04 LTS and later include Landlock support."

**Deep Agents Code** (different NemoClaw agent) [3]: Default: "`compatibility: strict`. Deep Agents sandbox startup fails closed when OpenShell cannot enforce the managed filesystem policy." Risk: "Silent Landlock degradation would leave the terminal coding harness with weaker filesystem isolation while still reporting a successful sandbox."

---

## References

[1] https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/reference/network-policies
[2] https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/security/security-controls/filesystem-controls
[3] https://docs.nvidia.com/nemoclaw/latest/user-guide/deepagents/security/security-controls/filesystem-controls
[4] https://docs.nvidia.com/openshell/latest/security/best-practices.html
