# Gateway Operations

## The Golden Rule
**NEVER kill the gateway process without immediately restarting it.**
**ALWAYS verify inference works after any restart before declaring success.**

## How the Gateway Runs
- Binary: `target/release/zeroclaw daemon --port 8080`
- MUST be started via `run.sh` — it extracts the Claude Code OAuth token from macOS Keychain
- Running `cargo run` or the binary directly = no `ANTHROPIC_OAUTH_TOKEN` = inference fails with "Agent request failed"

## Start the Gateway
```bash
cd ~/zeroclaw-main && ./run.sh daemon --port 8080 >> ~/.zeroclaw/logs/daemon.stdout.log 2>> ~/.zeroclaw/logs/daemon.stderr.log &
```
Then wait and verify:
```bash
for i in $(seq 1 24); do sleep 5; if lsof -i :8080 | grep -q zeroclaw; then echo "up after $((i*5))s"; curl -s http://localhost:8080/health; break; fi; echo "waiting... $((i*5))s"; done
```

## Verify Inference Works (REQUIRED after every restart)
```bash
curl -s -X POST http://localhost:8080/webhook \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer zc_local_dev_2026" \
  -d '{"message": "hello"}'
```
- Success: `{"response": "...", "model": "...", "tool_calls": []}`
- Failure: `{"error": "Agent request failed"}` = OAuth token missing, restart via run.sh

## Check if Running
```bash
lsof -i :8080 | grep zeroclaw
```

## Check Health
```bash
curl -s http://localhost:8080/health
```

## Why Inference Goes Down
1. **Stale OAuth token** — Claude Code session expired. Restart via run.sh to pick up fresh token.
2. **Started without run.sh** — Binary has no ANTHROPIC_OAUTH_TOKEN. Restart via run.sh.
3. **Port stolen** — Another process grabbed 8080 while zeroclaw was down. Kill it first, then restart.

## Check Logs
```bash
tail -f ~/.zeroclaw/logs/daemon.stdout.log
tail -f ~/.zeroclaw/logs/daemon.stderr.log
```

## The Automator App
- Located at `~/Desktop/ZeroClaw.app`
- Double-click to start gateway without a terminal
- Shows macOS notification on start or if already running
- Will NOT work if zeroclaw binary needs recompiling (use run.sh directly in that case)

## Architecture
```
Browser → ay8.app (Cloudflare Worker)
       → Next.js API routes
       → gateway.ay8.app (Cloudflare Tunnel)
       → localhost:8080 (ZeroClaw gateway)
       → Agent loop → Anthropic API
```
