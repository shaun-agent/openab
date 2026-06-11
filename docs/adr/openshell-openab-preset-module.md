# ADR: OpenShell OpenAB Preset Module

- **Status:** Discussion
- **Date:** 2026-06-10
- **Author:** OpenAB POC contributors
- **Related:** [OpenShell](../openshell.md), [ADR: OpenShell-Compatible Gateway WebSocket Authentication](./openshell-websocket-auth.md), [ADR: Custom Gateway](./custom-gateway.md)

---

## 1. Context

OpenAB supports deployment through Kubernetes-style infrastructure and through
OpenShell sandboxes. Kubernetes is operationally familiar to many platform
teams, but it is heavy for average clients. OpenShell gives OpenAB a stronger
agent sandbox boundary, but the current setup experience exposes too many
low-level details: sandbox image shape, provider credentials, network policy,
WebSocket policy, runtime tool installation paths, writable paths, and gateway
placement.

During the Google Chat + Kiro + OpenShell POC, the team validated the broad
shape but found the OpenShell route difficult to operate reliably without a
large amount of explicit configuration. The main blockers were:

- WebSocket credential rewriting and gateway authentication.
- Local proxy or bridge routing from the sandbox to the gateway.
- Kiro and model endpoint policy discovery.
- Runtime tool installation limitations.
- Provider credential values appearing as resolver placeholders rather than raw
  environment values.
- Runtime-vs-development expectations: the quick-start image must not be an
  OpenAB runtime image. It should prepare only a non-root writable workspace;
  OpenAB, `openab-agent`, and task-specific tools must be installed at runtime
  under `/sandbox`.
- Policy file compatibility: the OpenAB developer policy must be valid against
  the documented OpenShell CLI before the quick start can claim a one-shot
  setup.

This ADR proposes discussing an OpenAB-owned OpenShell preset module that makes
the common case simple while preserving OpenShell's security value.

## 2. Question

Is there a very simple, singular configuration that enables normal OpenAB agent
functionality, including web access and agent-installable tools, without
forcing average clients to understand every OpenShell policy primitive?

Today, based on the POC, the answer is effectively **no**. OpenShell does not
currently feel like it has a single "just make it work like Kubernetes" config
for OpenAB agents.

The closest product direction would be an opinionated preset, for example:

```text
openshell sandbox create --preset openab-full-agent
```

or:

```yaml
preset: openab-full-agent
network:
  mode: broad_web
tools:
  mode: runtime_user_local
filesystem:
  writable:
    - /sandbox
    - /sandbox/bin
    - /sandbox/.local
    - /sandbox/tmp
secrets:
  provider: openab
gateway:
  mode: host_or_managed
```

This should be treated as a product/API sketch, not a current OpenShell feature.

## 2.1 Current Testing Status

Status as of 2026-06-11: **runtime-bootstrap path documented; `openab-agent`
release packaging added for the next binary release**.

The current `docs/openshell.md` quick start is intended to prove the local
developer path:

```text
build neutral workspace image
create policy-governed sandbox
verify writable /sandbox paths
install openab at runtime into /sandbox/bin
install openab-agent at runtime into /sandbox/bin
write config
authenticate openab-agent
run openab through OpenShell
connect Discord bot
reply to a mention
```

It should not be read as proof that the sandbox has unrestricted root access,
nor that the developer policy is a production hardening profile.

Recent local E2E testing found:

- The bot can come online and reply when launched with `HOME=/sandbox` and a
  valid `openab-agent` auth file.
- The running sandbox is not suitable for system package installation.
  Attempts to install tools into `/usr/local/bin` or use `apt` fail under the
  non-root sandbox user, and that is an intended boundary.
- Most local users need an install-friendly sandbox, but install-friendly means
  runtime user-local installs under `/sandbox`, not a large image that
  preinstalls OpenAB, `openab-agent`, or every possible tool.
- Small standalone binaries can be staged under `/sandbox/bin`; larger tool
  trees should live under `/sandbox`; scratch downloads should use
  `/sandbox/tmp`.
- Host Codex auth (`~/.codex/auth.json`) is not interchangeable with
  `openab-agent` auth (`/sandbox/.openab/agent/auth.json`).
- The OpenShell default policy is not enough for a Codex/OpenAB runtime path.
  The quick start therefore needs an explicit developer policy that allows the
  OpenAB bootstrap, Discord, and model/auth endpoints.
- The repository publishes `openab` Linux release archives, and the release
  workflow should publish matching `openab-agent` archives as part of the same
  release. A quick-start run should stop if the selected version lacks either
  archive instead of baking missing binaries into the image.

Recommended docs stance while testing continues:

- Keep `docs/openshell.md` focused on the install-friendly local dev quick
  start.
- Include a concrete developer policy in the quick start because Discord,
  model/auth, and download traffic are part of the first successful run.
- If a policy file fails to apply during E2E, treat that as a policy/docs issue.
  Do not rewrite the policy in a private scratch file and call the guide
  successful.
- Do not count a run as successful if OpenAB or `openab-agent` was preinstalled
  in the image. The runtime install step is part of the product contract.

## 3. Core Difference

| Topic | Kubernetes Default Mental Model | OpenShell Default Mental Model |
|---|---|---|
| Agent rights by default | A pod/container can often run with broad outbound network, mounted env/secrets, writable container FS depending on image/user | Sandbox user, limited writable paths, network endpoints must be allowed, credentials resolved through provider model |
| Filesystem | Image FS plus writable container layer; can install tools if root/package manager is available | `/usr`, `/etc`, `/lib`, package-manager state are effectively not runtime-editable; writable paths are mainly `/sandbox` and `/tmp` |
| Network/web | Usually broad outbound unless NetworkPolicy/firewall restricts it | Network is policy-driven; endpoints often need allowlisting |
| Tool availability | Whatever is in image; can sometimes apt/pip/npm install at runtime | Keep the image neutral; install OpenAB, `openab-agent`, and extra tools at runtime into `/sandbox` |
| Secrets | Kubernetes Secrets/env/volumes are straightforward and raw values enter container | OpenShell provider values may appear as resolver handles, not raw env values; app must cooperate with resolution/rewrites |
| WebSocket/gateway | Usually direct networking between services/pods or host ingress | WebSocket policy/credential rewrite/proxy behavior can be tricky |
| Setup style | YAML-heavy but familiar: Deployment, Secret, Service, Ingress | Sandbox image + provider + policy + uploads + endpoint discovery |
| Failure mode | Misconfigured infra, image, ingress, RBAC | Policy blocks, credential resolution mismatch, non-writable paths, endpoint allowlist gaps |
| Average client UX | Complex, but many people know the pattern | Safer but less obvious; needs presets/templates to be approachable |
| "Enable everything" path | Run privileged/broad-network container if the operator accepts risk | Not really first-class; broad policy is possible in theory but cuts against OpenShell's design |

## 4. What Can Be Enabled

| Capability | Kubernetes | OpenShell |
|---|---|---|
| Full web browsing / HTTP calls | Usually yes by default | Possible, but needs broad or wildcard-like network policy |
| Google Chat webhook | Via ingress/service/gateway | Gateway likely still best outside sandbox or as managed component |
| Google API credentials | Mount secret/env directly | Prefer host gateway or provider-injected short-lived tokens |
| OpenAB runtime | Bake into image or install runtime | Install `openab` and `openab-agent` at runtime under `/sandbox/bin`; do not bake them into the quick-start image |
| CLI tools like `gws`, `aws`, `terraform`, `kubectl` | Bake into image or install runtime | Prefer agent-installed standalone tools under `/sandbox`; bake only neutral bootstrap dependencies |
| File editing | Whatever the container user can write | Mostly `/sandbox` unless policy/image is designed otherwise |
| Package installs | Easy if root + network + package manager | System package installs are not expected; use user-local binary/archive/`.deb` extraction patterns under `/sandbox` |
| Long-running bot | Deployment/restart policy | Possible, but more host/OpenShell lifecycle dependent |
| Strict per-agent security | Possible but requires Kubernetes hardening | Native strength of OpenShell |

## 5. Setup Shape

Kubernetes setup is heavier upfront but conceptually linear:

```text
build image
create secrets
deploy pod
expose service/ingress
set env
run
```

OpenShell setup, as seen in the POC, is more segmented:

```text
build neutral OpenShell-compatible workspace image
create provider credentials
create sandbox
set network policies
runtime-install OpenAB and openab-agent into /sandbox/bin
upload or write config/state
handle credential rewrite/resolution
handle gateway bridge/proxy
run OpenAB
discover blocked endpoints
iterate
```

The OpenShell path is harder for average clients unless OpenAB or OpenShell
ships a blessed preset.

## 6. Proposed Presets

For OpenAB productization, average clients should not need to understand
OpenShell policy primitives. OpenAB should expose a small number of presets.

| Preset | Purpose | Default Rights |
|---|---|---|
| `safe-agent` | Enterprise/security-sensitive deployments | Narrow endpoints, no runtime installs, secrets via provider |
| `web-agent` | Most normal OpenAB bots | Broad HTTPS/WebSocket outbound, neutral workspace image, agent-installable `/sandbox` |
| `dev-agent` | Debug/setup only | Broader network, runtime OpenAB/agent bootstrap under `/sandbox`, short-lived, not production webhook |

The most useful default is probably `web-agent`: broad web access, a neutral
workspace image, writable `/sandbox`, runtime user-local installs, and a managed
gateway path. This gets close to Kubernetes convenience without pretending the
sandbox has no security boundary.

### 6.1 Policy Recommendations

These recommendations are a product direction. The local `dev-agent` policy in
`docs/openshell.md` is the first concrete policy artifact and should be kept
valid against the documented OpenShell CLI.

| Tier | Use case | Network posture | Install posture | Docs posture |
|---|---|---|---|---|
| `dev-agent` | First successful local OpenAB bot and normal developer use | Broad enough for OpenAB release downloads, Discord, and model/auth endpoints; add more endpoints deliberately from logs | Runtime install `openab`, `openab-agent`, and extras under `/sandbox`; no system package installs | `docs/openshell.md` quick start |
| `web-agent` | Normal deployed OpenAB assistant with web/API tools | Broad HTTPS/WebSocket egress for selected providers and tools | Neutral workspace image; agent installs extras under `/sandbox` | Policy-specific docs after validation |
| `runtime-agent` | Smaller production-like bot | Only what the image and current OpenShell defaults require to connect Discord and model APIs | Minimal workspace image; optional user-local `/sandbox/bin` installs if policy permits | Advanced/runtime note |
| `safe-agent` | Enterprise/security-sensitive deployment | Narrow endpoint allowlist per agent/provider | No runtime installs | Production hardening guide |

Policy implementation guidance:

- Validate policy files against the documented OpenShell CLI before calling them
  stable.
- Prefer an OpenAB-owned policy artifact after CI or release testing proves it
  applies cleanly with the supported OpenShell version.
- Keep broad policy files labeled as developer policies until they are narrowed
  and validated for production.
- Treat policy edits made during E2E as findings, not as test harness fixes.
- Separate network policy from install policy. A sandbox can have broad egress
  and still be intentionally non-root while allowing user-local installs under
  `/sandbox`.
- Track `openab-agent` release packaging as a first-class dependency of the
  no-preinstall quick start. Each supported OpenShell quick-start version must
  publish both `openab` and `openab-agent` Linux archives.

## 7. Preset Responsibilities

An `openab-full-agent` or `web-agent` preset should own the following decisions:

- Build or select an OpenAB-compatible sandbox image.
- Set `HOME=/sandbox`.
- Make `/sandbox`, `/sandbox/.local`, `/sandbox/.cache`, and `/tmp` writable.
- Keep system paths read-only.
- Run as the non-root `sandbox` user.
- Keep the image neutral. Do not preinstall OpenAB, `openab-agent`, model CLIs,
  language toolchains, or task-specific tools.
- Install OpenAB, `openab-agent`, and extra tools into `/sandbox` using the
  documented user-local pattern.
- Enable broad outbound HTTPS for normal web/model/API access.
- Enable WebSocket egress for OpenAB gateway connectivity.
- Provide a clear gateway mode: host gateway, managed gateway, or direct
  platform connection.
- Provide a provider/secret convention that avoids raw long-lived secrets in
  sandbox config.
- Preserve an escape hatch for stricter endpoint allowlists.

## 8. Product Positioning

The product question is not "can OpenShell safely expose everything?" It is:

```text
Can OpenShell offer a one-command default that is permissive enough for normal
OpenAB agents, while still safer than a random Kubernetes pod?
```

That would be strong positioning. Kubernetes can run the same workload, but the
operator must add the security boundary deliberately. OpenShell starts with the
security boundary, but needs an accessible preset that makes the common OpenAB
agent path feel obvious.

## 9. Decision To Discuss

OpenAB should consider an OpenShell preset module with three layers:

1. A blessed neutral workspace image family with writable user-local install
   paths and no preinstalled OpenAB runtime.
2. A small preset policy vocabulary such as `safe-agent`, `web-agent`, and
   `dev-agent`.
3. A gateway and credential pattern that handles webhook platforms without
   forcing clients to debug WebSocket credential rewrite behavior.

The immediate proposal is not to remove OpenShell's explicit policy model. The
proposal is to hide the routine policy decisions behind product-level presets
and leave the low-level controls for operators who need them.
