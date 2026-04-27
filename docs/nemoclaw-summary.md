# NemoClaw Edge Agent -- Project Summary

## Hardware

- **Server:** Cisco UCS C220 M5 running Proxmox with an LXC container for the agent stack
- **GPUs:** 2x NVIDIA Tesla T4 (16GB VRAM each), passed through to the LXC container
- **OS:** Debian (inside Proxmox LXC)
- **Container Runtime:** Docker CE with NVIDIA Container Toolkit for GPU passthrough
- **Network:** Private LAN IP, accessible from local Mac

---

## Container Stack

Three Docker containers run on the host:

### 1. Ollama (LLM Inference)

```bash
# /opt/ollama/docker-compose.yml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=-1
      - OLLAMA_FLASH_ATTENTION=1
      - OLLAMA_NUM_PARALLEL=4
      - OLLAMA_MAX_LOADED_MODELS=2
    volumes:
      - ollama-data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
volumes:
  ollama-data:
```

Pull the model after starting:

```bash
docker exec ollama ollama pull qwen3:8b
```

### 2. CML MCP Server (Cisco Modeling Labs Tools)

```bash
docker run -d \
  -p 3010:3010 \
  -e DISABLE_JWT_AUTH=true \
  -e CML_HOST=<CML_HOST> \
  -e CML_USERNAME=<CML_USERNAME> \
  -e CML_PASSWORD=<CML_PASSWORD> \
  -e CML_VERIFY_SSL=false \
  --name cml-mcp-server \
  --restart unless-stopped \
  ghcr.io/presidio-federal/cml-mcp:latest
```

CML credentials are baked into the container as environment variables because OpenClaw has a known bug where `streamable-http` transport does not forward custom headers to remote MCP servers (openclaw/openclaw#65590).

### 3. NemoClaw (OpenShell Cluster)

Managed by the `nemoclaw` CLI. The container is `openshell-cluster-nemoclaw`.

```bash
# Install NemoClaw
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash

# Onboard with Ollama + Qwen
NEMOCLAW_PROVIDER=ollama NEMOCLAW_MODEL=qwen3:8b nemoclaw onboard --non-interactive
```

---

## NemoClaw Post-Onboard Configuration

After `nemoclaw onboard`, several changes must be made from the **host** via `docker exec` because the sandbox has a Landlock read-only policy on `/sandbox/.openclaw/`. This is a known bug (NVIDIA/NemoClaw#719) -- `openclaw.json` is created as root:root 444, and Landlock enforces `/sandbox/.openclaw` as read-only.

### Fix config permissions (required after every onboard)

Use `kubectl exec` inside the cluster container to bypass Landlock:

```bash
docker exec openshell-cluster-nemoclaw kubectl exec -n openshell <SANDBOX_NAME> -c agent -- \
  sh -c "chown -R sandbox:sandbox /sandbox/.openclaw/ && chmod 700 /sandbox/.openclaw/ && chmod 600 /sandbox/.openclaw/openclaw.json"
```

If `kubectl exec` doesn't work, edit the config via `sed` on the PVC path. Find the path first:

```bash
docker exec -u 0 openshell-cluster-nemoclaw find / -name "openclaw.json" 2>/dev/null
```

The persistent config is at:
```
/var/lib/rancher/k3s/storage/pvc-<UUID>_openshell_workspace-<SANDBOX_NAME>/.openclaw/openclaw.json
```

### Remove the `configWrites` bad key

Every fresh onboard creates an invalid `configWrites` key under `channels.defaults` that prevents the gateway from starting:

```bash
docker exec -u 0 openshell-cluster-nemoclaw sed -i '/"configWrites"/d' \
  /var/lib/rancher/k3s/storage/pvc-<UUID>_openshell_workspace-<SANDBOX_NAME>/.openclaw/openclaw.json
```

### Set gateway bind to LAN

By default the gateway binds to `127.0.0.1` (loopback only). To access the UI from another machine:

```bash
docker exec -u 0 openshell-cluster-nemoclaw sed -i 's/"bind": "loopback"/"bind": "lan"/' \
  /var/lib/rancher/k3s/storage/pvc-<UUID>_openshell_workspace-<SANDBOX_NAME>/.openclaw/openclaw.json
```

### Set allowed origins for the Control UI

Without this, the UI shows "origin not allowed":

```bash
# From inside the sandbox (if permissions are fixed), or via docker exec sed
openclaw config set gateway.controlUi.allowedOrigins '["http://<SERVER_IP>:18789"]'
openclaw config set gateway.controlUi.allowInsecureAuth true
```

### Add MCP server config

The correct config key is `mcp.servers` (NOT `mcpServers`):

```bash
openclaw config set mcp.servers.cml-mcp --json '{"url": "http://host.openshell.internal:3010/mcp", "transport": "streamable-http"}'
```

### Start the port forward

```bash
openshell forward start 0.0.0.0:18789 <SANDBOX_NAME> -d
```

Then access the UI at `http://<SERVER_IP>:18789/#token=<TOKEN_FROM_ONBOARD_OUTPUT>`.

### Restart the gateway inside the sandbox

The gateway must run in the foreground in a container (`openclaw gateway restart` does not work):

```bash
nemoclaw <SANDBOX_NAME> connect
# Inside the sandbox:
openclaw gateway &
```

---

## Skills Installation

### Correct method

Skills are directories containing a `SKILL.md` file with YAML frontmatter. From the **host**:

```bash
# SCP zipped skills to the server into a persistent location
scp my-skill.zip root@<SERVER_IP>:/root/skills/

# On the server, unzip
mkdir -p /root/skills
cd /root/skills
unzip my-skill.zip -d my-skill

# Install into sandbox using the nemoclaw CLI
nemoclaw <SANDBOX_NAME> skill install /root/skills/my-skill/
```

The `nemoclaw skill install` command handles uploading the skill into the sandbox, setting permissions, and triggering agent re-discovery. After installing, start a new session (`/new` in the UI) so OpenClaw picks up the skills.

### What went wrong

1. **Wrong paths -- repeatedly:** Instead of using `nemoclaw skill install` from the host (the documented method), skills were manually placed at multiple wrong locations inside the sandbox: `~/skills/`, `/sandbox/skills/`, `~/.openclaw/skills/`, `/sandbox/.openclaw/skills/`. None of these are where the agent actually scans for prompt injection.

2. **The correct workspace path:** OpenClaw scans `~/.openclaw/workspace/skills/` (highest precedence). The `nemoclaw skill install` command places skills there automatically. When skills were manually placed elsewhere, they showed as "ready" in `openclaw skills list` but were never injected into the agent's prompt (OpenClaw bug openclaw/openclaw#18989).

3. **Model too small for skills:** Even after skills were correctly placed (confirmed via `systemPromptReport`), the initial model (Nemotron-mini) was too small to utilize them. The system prompt was ~30K characters with 23 tools and 9 skills. Switching to Qwen3:8b resolved this.

---

## Known Bugs and Workarounds

| Bug | Impact | Workaround |
|-----|--------|------------|
| **NemoClaw #719**: `openclaw.json` created as root:root 444 + Landlock read-only | Cannot change any config from inside the sandbox (skills, MCP, origins, bind, model) | Fix permissions via `kubectl exec` or edit the PVC file directly with `docker exec sed` |
| **NemoClaw #719**: `configWrites` invalid key in `channels.defaults` | Gateway refuses to start | Remove the key with `sed -i '/"configWrites"/d'` on the PVC config file |
| **OpenClaw #65590**: `streamable-http` transport does not forward custom headers | Cannot pass auth headers (e.g., CML credentials) from OpenClaw to remote MCP servers | Bake credentials into the MCP server container as environment variables instead |
| **OpenClaw #18989**: Custom skills in `~/.openclaw/skills/` not injected into prompt | Agent doesn't know about installed skills even though they show as "ready" | Place skills in `~/.openclaw/workspace/skills/` instead |
| **OpenClaw UI model dropdown**: Shows 800+ models from all providers | Confusing when trying to select local Ollama model | Use `/model ollama-local/qwen3:8b` in the chat instead of the dropdown |
| **`openclaw gateway restart`**: Does not work in containers | Gateway won't restart | Use `openclaw gateway &` (foreground, backgrounded) instead |
| **Port forward binds to 127.0.0.1**: Default `openshell forward start` only listens on loopback | Cannot access UI from another machine on the LAN | Use `openshell forward start 0.0.0.0:18789 <name> -d` |

---

## Architecture Summary

```
Mac (browser) --> http://<SERVER_IP>:18789 --> OpenShell Port Forward
  --> NemoClaw Sandbox (OpenClaw Gateway on 0.0.0.0:18789)
    --> Ollama (qwen3:8b) via host.openshell.internal:11434
    --> CML MCP Server via host.openshell.internal:3010
```

All inference stays on-prem. The NemoClaw sandbox enforces network policy, filesystem isolation (Landlock), and process isolation (seccomp). Ollama runs on the Tesla T4 GPUs. The CML MCP server connects to Cisco Modeling Labs for factory simulation tooling.
