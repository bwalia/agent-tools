# Ollama lockdown task (2026-10-10)

Work is done in the infra repo (`~/projects/diy-tax-return-infra`) on branch
`feat/ollama-lockdown`, checked out as a git worktree at `./infra-worktree`.

## Original prompt

- lon1 (195.20.255.201) is the wslproxy edge that int-opsapi resolves to.
- 192.168.1.36 is k3s-worker-08 in ansible/inventory.yml.
- Ollama runs on this Mac (192.168.1.177) from the user startup file ~/Library/LaunchAgents/com.ollama.server.plist.
- Nothing in the infra repo manages Ollama or ollama.workstation.co.uk yet, so the prompt asks it to add that.

Lock down and harden the shared Ollama server in this infra repo. Today
(2026-10-10) it was overwhelmed, and opsapi's page assistant on int shows
"Sorry — the assistant is unavailable right now."

### What happened (evidence)
- Ollama runs on a Mac mini/MacBook at LAN IP 192.168.1.177, port 11434, started by the
  launchd agent ~/Library/LaunchAgents/com.ollama.server.plist with OLLAMA_HOST=0.0.0.0:11434,
  OLLAMA_ORIGINS=*, OLLAMA_NUM_PARALLEL=4, OLLAMA_MAX_LOADED_MODELS=5, OLLAMA_CONTEXT_LENGTH=16384,
  OLLAMA_KEEP_ALIVE=2m, OLLAMA_FLASH_ATTENTION=1, OLLAMA_KV_CACHE_TYPE=q8_0. OLLAMA_MAX_QUEUE unset (512).
- Public IPs called Ollama directly with no auth: 84.247.152.177 (Contabo, ~470 POST
  /v1/chat/completions at ~16:50), 66.132.195.x (scanners), 38.137.21.140. The queue filled,
  so every caller got `503 server busy ... maximum pending requests exceeded`, including opsapi int
  (from 193.237.176.232).
- Legitimate callers: 127.0.0.1 (KubePilot, llama3.2:3b-dash), 192.168.1.36 (k3s-worker-08,
  1.5–3 min /v1/chat/completions), opsapi int/prod via https://ollama.workstation.co.uk
  (qwen3.8:latest, /api/chat).
- Port 11434 is reachable from the internet: router port-forward or edge pass-through. Find out which.

### Do this
1. Inventory (`ai_hosts` / `ollama-mac`) and docs/runbooks/ollama.md (callers, traced public path, restart).
2. ansible/ollama-host.yml: pf anchor /etc/pf.anchors/ollama allowing 11434 only from 127.0.0.1,
   192.168.1.0/24, the WireGuard subnets in group_vars, and the edge/tunnel address. Idempotent, --check friendly.
   Router port-forward → manual step in the runbook.
3. wslproxy vhost for ollama.workstation.co.uk via new-vhost.py: Bearer key from Vault, ~30 req/min/IP,
   upstream over LAN/WG, timeouts ≥300 s, path allowlist (/api/chat, /api/tags, /api/version, /api/ps,
   /v1/chat/completions, /api/embed). Apply via ansible/wslproxy-vhosts.yml.
4. Manage the launchd plist (back it up first): OLLAMA_MAX_QUEUE=32, keep NUM_PARALLEL=4, drop ORIGINS=*,
   keep HOST=0.0.0.0:11434. Restart via launchctl kickstart only after confirmation.
5. opsapi fallback: AI_FALLBACK_PROVIDER=anthropic, AI_FALLBACK_MODEL=claude-sonnet-5-5,
   ANTHROPIC_API_KEY + OLLAMA_API_KEY from Vault, wherever opsapi int/prod values live
   (environments/, gitops/). Don't touch opsapi nginx `env`.

### Rules
- No secrets in git. `make lint` before commit. Branch + PR. No applying to hosts or the edge
  until the PR is approved (read-only discovery only). Never restart Ollama or reload wslproxy without asking.

### Verify
- External curl to <public-ip>:11434 times out or is refused; hostname without key → 401/403, with key → 200;
  local /api/chat → 200; opsapi int agent status "up"; a 100-request burst gets 429s and the queue stays ≤ OLLAMA_MAX_QUEUE.
