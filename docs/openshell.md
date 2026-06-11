# OpenShell Quick Start

Run one OpenAB Discord bot inside an [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell) sandbox.

This guide is optimized for the first useful OpenAB agent sandbox:

- The sandbox image is not an OpenAB runtime image.
- The image does not preinstall OpenAB, `openab-agent`, language toolchains, cloud CLIs, or task-specific tools.
- Runtime binaries are installed by the agent into `/sandbox/bin`.
- The running agent uses `/sandbox` as `HOME`, working directory, cache, and scratch space.

Install-friendly does **not** mean root access or `apt-get` inside the running sandbox. It means the agent can download and install user-local tools under `/sandbox` without rebuilding the image.

## What You Will Get

By the end, you should have:

- One OpenShell sandbox named `oab`
- A writable agent home at `/sandbox`
- Runtime install paths under `/sandbox/bin` and `/sandbox/.local`
- `openab` installed at runtime under `/sandbox/bin`
- `openab-agent` installed at runtime under `/sandbox/bin`
- One Discord bot connected to Discord
- One `openab-agent` session using ChatGPT/Codex subscription auth
- A bot that replies when mentioned in Discord

## Prerequisites

- Docker is running on the host
- OpenShell CLI is installed
- You have a Discord bot token
- You have a Discord channel ID for the first test
- You have a ChatGPT account that can authenticate `openab-agent`

Install OpenShell:

```bash
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
```

If OpenShell cannot talk to Docker, add your user to the `docker` group and start a new login session:

```bash
sudo usermod -aG docker "$USER"
# Log out and back in, or run: loginctl terminate-user "$USER"
```

## 1. Set Host Inputs

Keep secrets in your shell environment. Do not paste tokens into `config.toml`.

```bash
export DISCORD_BOT_TOKEN="your-discord-bot-token"
export DISCORD_CHANNEL_ID="your-discord-channel-id"
export OPENAB_VERSION="0.8.5-beta.8"
```

## 2. Build The Workspace Image

Build the OpenShell-compatible workspace image from this repo:

```bash
docker build -t oab-workspace -f openshell/Dockerfile .
```

This image only prepares the non-root `sandbox` user, a writable `/sandbox` layout, and the bootstrap utilities needed to fetch release archives. It does not include OpenAB, `openab-agent`, Codex, Go, Node, Python, Rust, cloud CLIs, or task-specific tools.

## 3. Create A Network Policy

OpenShell is policy-oriented. The default policy can block Discord, model calls, GitHub release downloads, and other runtime installs. Create an explicit local developer policy before relying on the sandbox.

From the host:

```bash
cat > openab-dev-agent-policy.yaml <<'EOF'
version: 1
filesystem_policy:
  include_workdir: true
  read_only:
    - /bin
    - /usr
    - /lib
    - /lib64
    - /etc
    - /dev/urandom
  read_write:
    - /sandbox
    - /tmp
    - /dev/null
landlock:
  compatibility: best_effort
process:
  run_as_user: sandbox
  run_as_group: sandbox
network_policies:
  openab_runtime_bootstrap:
    name: OpenAB runtime bootstrap downloads
    endpoints:
      - host: github.com
        port: 443
      - host: api.github.com
        port: 443
      - host: objects.githubusercontent.com
        port: 443
      - host: release-assets.githubusercontent.com
        port: 443
    binaries:
      - path: /usr/bin/curl
      - path: /usr/bin/tar
      - path: /sandbox/bin/**
      - path: /sandbox/.local/bin/**
  discord_runtime:
    name: Discord gateway and REST API
    endpoints:
      - host: discord.com
        port: 443
      - host: gateway.discord.gg
        port: 443
        protocol: websocket
    binaries:
      - path: /sandbox/bin/openab
      - path: /sandbox/bin/openab-agent
      - path: /sandbox/bin/**
  openai_runtime:
    name: OpenAI and Codex auth/runtime
    endpoints:
      - host: api.openai.com
        port: 443
      - host: auth.openai.com
        port: 443
      - host: chatgpt.com
        port: 443
      - host: ab.chatgpt.com
        port: 443
    binaries:
      - path: /sandbox/bin/openab-agent
      - path: /sandbox/bin/**
EOF
```

This policy is intentionally a developer policy, not a production hardening profile. For production, narrow endpoints and binaries after observing denied-request logs.

## 4. Create The Sandbox

```bash
openshell sandbox create --name oab \
  --from oab-workspace:latest \
  --policy openab-dev-agent-policy.yaml \
  --env HOME=/sandbox \
  --env PATH=/sandbox/bin:/sandbox/.local/bin:/usr/local/bin:/usr/bin:/bin \
  --env TMPDIR=/sandbox/tmp \
  --env OPENAB_VERSION="$OPENAB_VERSION" \
  --env DISCORD_CHANNEL_ID="$DISCORD_CHANNEL_ID" \
  -- sleep infinity
```

Reconnect later with:

```bash
openshell sandbox connect oab
```

## 5. Verify Runtime Install Paths

Inside the sandbox:

```bash
export HOME=/sandbox
export PATH="/sandbox/bin:/sandbox/.local/bin:$PATH"
export TMPDIR=/sandbox/tmp

test -w /sandbox
mkdir -p /sandbox/bin /sandbox/.local/bin /sandbox/.cache /sandbox/tmp
printf '#!/bin/sh\necho openab-local-install-ok\n' > /sandbox/bin/openab-local-install-test
chmod +x /sandbox/bin/openab-local-install-test
openab-local-install-test
```

Expected output:

```text
openab-local-install-ok
```

## 6. Install OpenAB Runtime Binaries

Inside the sandbox, install OpenAB into `/sandbox/bin`:

```bash
set -eu
cd /sandbox/tmp
ARCH="$(uname -m)"
case "$ARCH" in
  x86_64) OPENAB_ARCH="x64" ;;
  aarch64|arm64) OPENAB_ARCH="arm64" ;;
  *) echo "unsupported architecture: $ARCH" >&2; exit 1 ;;
esac

curl -fsSL \
  -o openab.tar.gz \
  "https://github.com/openabdev/openab/releases/download/v${OPENAB_VERSION}/openab-${OPENAB_VERSION}-linux-${OPENAB_ARCH}.tar.gz"
tar -xzf openab.tar.gz
install -m 0755 openab /sandbox/bin/openab
openab --help >/dev/null
```

Install `openab-agent` into `/sandbox/bin`:

```bash
set -eu
cd /sandbox/tmp
curl -fsSL \
  -o openab-agent.tar.gz \
  "https://github.com/openabdev/openab/releases/download/v${OPENAB_VERSION}/openab-agent-${OPENAB_VERSION}-linux-${OPENAB_ARCH}.tar.gz"
tar -xzf openab-agent.tar.gz
install -m 0755 openab-agent /sandbox/bin/openab-agent
openab-agent --help >/dev/null
```

If the `openab-agent` archive is missing for the selected version, stop and use a version that publishes it. Do not fix the quick start by baking `openab-agent` into the image.

## 7. Create The OpenAB Config

Inside the sandbox:

```bash
cd /sandbox

cat > config.toml <<EOF
[discord]
bot_token = "\${DISCORD_BOT_TOKEN}"
allow_all_channels = false
allowed_channels = ["${DISCORD_CHANNEL_ID}"]
allow_all_users = true
allow_dm = false
message_processing_mode = "per-thread"

[agent]
command = "openab-agent"
working_dir = "/sandbox"

[agent.env]
HOME = "/sandbox"
PATH = "/sandbox/bin:/sandbox/.local/bin:/sandbox/.cargo/bin:/usr/local/bin:/usr/bin:/bin"
TMPDIR = "/sandbox/tmp"
OPENAB_AGENT_OPENAI_MODEL = "gpt-5.4-mini"

[pool]
max_sessions = 1
session_ttl_hours = 1

[reactions]
enabled = true
remove_after_reply = false
EOF
```

Why these defaults:

- `bot_token = "${DISCORD_BOT_TOKEN}"` keeps the secret out of the file.
- `allowed_channels` limits the first bot run to one channel.
- `working_dir = "/sandbox"` gives the agent a writable project directory.
- `HOME = "/sandbox"` keeps auth at `/sandbox/.openab/agent/auth.json`.
- `PATH` makes runtime-installed tools available to OpenAB and the agent.
- `TMPDIR = "/sandbox/tmp"` keeps scratch work inside the writable sandbox home.

## 8. Authenticate The Agent

Inside the sandbox:

```bash
HOME=/sandbox openab-agent auth codex-oauth --no-browser
```

Open the printed URL in your browser. After approval, the browser may try to redirect to `localhost:1455`. Copy the full callback URL from the browser address bar and paste it back into the terminal.

Verify auth:

```bash
HOME=/sandbox openab-agent auth status
```

Expected result: the status command reports a valid Codex/OpenAI auth file under `/sandbox/.openab/agent/auth.json`.

## 9. Run The Active OpenAB Agent

From the host, start the long-running OpenAB process inside the OpenShell sandbox:

```bash
: "${DISCORD_BOT_TOKEN:?set DISCORD_BOT_TOKEN first}"

openshell sandbox exec -n oab \
  --env DISCORD_BOT_TOKEN="$DISCORD_BOT_TOKEN" \
  --env DISCORD_CHANNEL_ID="$DISCORD_CHANNEL_ID" \
  --workdir /sandbox \
  --timeout 0 \
  -- sh -lc 'exec env HOME=/sandbox USER=sandbox LOGNAME=sandbox TMPDIR=/sandbox/tmp PATH=/sandbox/bin:/sandbox/.local/bin:/sandbox/.cargo/bin:/usr/local/bin:/usr/bin:/bin openab run -c /sandbox/config.toml'
```

Expected log lines:

```text
config loaded agent_cmd=openab-agent
discord bot running
discord bot connected user=<your-bot-name>
registered global slash commands
```

Mention the bot in the configured Discord channel. It should reply in the channel or thread.

## Installing Extra Tools At Runtime

Use the OpenAB home-directory install pattern:

```bash
mkdir -p /sandbox/bin /sandbox/tmp
curl -fsSL -o /sandbox/bin/<tool> "<official-linux-binary-url>"
chmod +x /sandbox/bin/<tool>
export PATH="/sandbox/bin:$PATH"
<tool> --version
```

For tools distributed as archives, download to `/sandbox/tmp`, extract there, and copy only the final executable into `/sandbox/bin`.

For tools distributed as `.deb` packages, extract without root:

```bash
mkdir -p /sandbox/bin /sandbox/tmp/deb-extract
curl -fsSL -o /sandbox/tmp/package.deb "<deb-url>"
dpkg-deb -x /sandbox/tmp/package.deb /sandbox/tmp/deb-extract
cp /sandbox/tmp/deb-extract/usr/bin/<binary> /sandbox/bin/
chmod +x /sandbox/bin/<binary>
rm -rf /sandbox/tmp/package.deb /sandbox/tmp/deb-extract
```

Rules for agents:

- Do not use `sudo`.
- Do not write to `/usr`, `/opt`, or `/usr/local/bin`.
- Install binaries to `/sandbox/bin`.
- Install larger user-local tool trees under `/sandbox`.
- Use `/sandbox/tmp` for scratch work.
- Detect architecture before downloading binaries.
- Verify every install with `<tool> --version` or equivalent.
- If OpenShell blocks a required download endpoint, inspect `openshell logs oab --source sandbox` and update the policy deliberately.

See [Agent-Installable Tools](agent-installable-tools.md) for the general pattern.

## Troubleshooting

| Symptom | Check | Fix |
| --- | --- | --- |
| `failed to query Docker daemon version` | Docker access | Add user to `docker` group and start a new login session |
| Policy rejects at create time | `openshell sandbox create` error | Fix `openab-dev-agent-policy.yaml`; do not continue with a private untracked policy |
| Download blocked | `openshell logs oab --source sandbox` | Add the required host and binary path to the policy |
| Bot token error | `test -n "$DISCORD_BOT_TOKEN" && echo set` | Re-export the token in the shell that starts OpenAB |
| Auth file searched under `/root` | Log says `/root/.openab/...` | Run with `HOME=/sandbox` |
| Bot online but no reply | `openab-agent auth status` | Re-run `openab-agent auth codex-oauth --no-browser` |
| Tool install says `/usr/local/bin` or `apt` is not writable | The running sandbox user is non-root | Install to `/sandbox/bin` or another `/sandbox` path |
| Tool requires system libraries not present in the image | Binary exists but fails at launch | Choose a static upstream binary, extract compatible `.deb` dependencies into `/sandbox`, or build a deployment-specific image only after documenting the exception |

## E2E Test Rules

When testing this guide with an agent, keep the test honest:

- Use `openshell/Dockerfile`.
- Treat the guide author and the E2E test subject as separate roles.
- Do not preinstall OpenAB, `openab-agent`, language toolchains, or task-specific tools into the image.
- If the test subject hits a build or setup failure, let the test subject diagnose, edit, rebuild, and report the fix.
- Do not silently switch to a different auth format.
- Do not copy host `~/.codex/auth.json` into `/sandbox/.openab/agent/auth.json`; `openab-agent` has its own auth file shape.
- If auth is unavailable, stop at the auth step and report that browser login is required.
- If a policy file fails to apply, report a docs/policy compatibility issue instead of generating a private fixed policy.
- Prove runtime installability by installing `openab` and `openab-agent` under `/sandbox/bin`.
- Prove the sandbox has an active agent by starting `openab run` through `openshell sandbox exec`.
- Always delete disposable test sandboxes and scratch files after the run.

## Cleanup

```bash
openshell sandbox delete oab
rm -f openab-dev-agent-policy.yaml
```

## Advanced Reading

- [Agent-Installable Tools](agent-installable-tools.md) — runtime install pattern for tools under the agent home directory.
- [OpenShell OpenAB preset module ADR](adr/openshell-openab-preset-module.md) — future `safe-agent`, `web-agent`, and `dev-agent` presets.
- [Native Agent](native-agent.md) — `openab-agent` auth and model options.
- [Secrets Management](secrets-management.md) — production secret patterns.
