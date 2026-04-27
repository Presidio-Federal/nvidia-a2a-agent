# Factory Edge Agent — checklist

Track progress with `- [ ]` / `- [x]` in this file (GitHub Flavored Markdown task lists).

---

## AI Studio

- [ ] **A2A connectivity and testing** — reach remote agent card (`/.well-known/agent.json`), bearer token auth, and error paths from cloud to tunnel endpoint.
- [ ] **Workflow testing** — verify an AI Studio agent can delegate multi-step tasks to the remote agent via A2A and receive structured results end-to-end.
- [ ] **Document operator runbook** — URLs (`a2a.vanginkel.tech` / agent UI), Cloudflare Access expectations, and who to contact for token rotation.
- [ ] **Agent registry alignment** — confirm remote agent capabilities in AI Studio match what the edge agent advertises (tools, skills, limits) to avoid silent delegation failures.
- [ ] **Post–AI Studio upgrade smoke test** — re-run A2A delegation after any platform or Network Ops agent template change.

---

## Remote Agent

- [ ] **Model testing** — migrate from Qwen 2.5 to a newer **Nemotron** (or other) model with a **larger context window**; validate tool-calling quality and latency on target GPUs.
- [ ] **Container migration** — package and deploy **additional MCP containers** on the remote host (images, env, ports, restart policy, health checks).
- [ ] **Workflow testing** — continue **local workflow** runs for **performance** (router vs specialist latency, prompt sizes, LangSmith traces if used).
- [ ] **Package agent for redeploy** — reproducible artifact: code + `nat-config.yml` + `.env` template + systemd units + documented install order for a clean host.
- [ ] **NVIDIA GPU visibility and optimization** — integrate extra NVIDIA tooling (e.g. metrics in observability path, `nvidia-smi`/DCGM exposure, or NAT/UI hooks) so operators can see utilization and tune batching / concurrency.
- [ ] **nat-config / env parity** — after any model or backend change, confirm `OLLAMA_MODEL` (or vLLM equivalent), `nat-config.yml`, and restart procedure are documented and tested together.
- [ ] **Expand MCP + skills** — wire any remaining Cisco ecosystem servers (Splunk, ThousandEyes, Meraki, etc.) only where needed for Cisco Live story; keep per-specialist context small per README guidance.
- [ ] **Cisco Live demo dry run** — scripted asks (CML, IOS-XE, general) with timing notes and fallback if tunnel or upstream is slow.

---

## Hardware

- [ ] **Wipe Proxmox and install Debian bare metal** — plan: firmware/RAID, disk layout, static networking, SSH hardening, and **reinstall** NVIDIA driver + Container Toolkit on the new base OS.
- [ ] **Migrate from Ollama to vLLM** — for better throughput/latency at scale; define serving API, model weights location, GPU memory headroom, and how NAT / LangGraph will call the new endpoint.
- [ ] **Validate GPU passthrough vs bare metal** — confirm no performance regression from previous LXC/Proxmox path; document final topology for auditors.
- [ ] **Backups and secrets** — secure copy of non-secret config templates + procedure for `.env` / tokens **without** committing secrets to git.
- [ ] **Power and thermal check** — sustained GPU load during vLLM + MCP demo window (Cisco Live booth reliability).

---

## Cross-cutting (optional but useful)

- [ ] **LangSmith / tracing** — ensure production vs demo traces are separated and sampling does not leak sensitive prompts.
- [ ] **Security review** — Cloudflare Access policies, A2A bearer token storage, and MCP container network exposure (localhost-only where possible).
- [ ] **Rollback plan** — one documented path to revert model or inference backend if demo-week behavior regresses.
