# Giving the assistant access to Railway logs + volume

## Why this exists

The Railway API (`backboard.railway.com`) is unreachable from the assistant's
sandbox (its network policy returns 403 for that host — the same reason exchange
APIs are blocked there). So the Railway CLI can't run *from the chat session*.

The fix: run the Railway CLI **on GitHub's runners** (open internet, token stored
as a secret) and commit the results into this repo, where the assistant already
has read access. `.github/workflows/railway-sync.yml` does that on a 6-hour
schedule (and on demand). Output lands in
`starters/quant-research-loop/railway-sync/`:

- `deploy-logs.txt` — the service's last 400 log lines (`[boot]`/`[check]`/`[refresh]`)
- `scoreboard.json`, `quant-forward-state.md`, `quant-forward-log.md` — pulled from the Volume
- `status.txt`, `volumes.json`, `volume-files.json` — token/volume diagnostics

## One-time setup (you do this in GitHub)

1. **Repo → Settings → Secrets and variables → Actions → New repository secret**
   - Name: `RAILWAY_TOKEN`
   - Value: a Railway **Project Token** (Railway → your project → Settings → Tokens →
     Create Token → Project token, environment = production).
2. **Same page → Variables tab → New repository variable** (recommended)
   - Name: `RAILWAY_SERVICE`
   - Value: the service name (Railway dashboard → the service card's name, e.g.
     `loop-engineering`). Skip only if the project has exactly one service.
3. **Actions tab → "Railway Sync" → Run workflow** (branch: `main`) to run it now,
   or wait for the schedule.

After it runs, the assistant reads `railway-sync/deploy-logs.txt` and can diagnose
the service (e.g. the `[refresh] live fetch failed (...)` line that explains why a
strategy is stuck on "awaiting data").

> The repo is public and these logs are benign (no secrets printed by the service).
> The `RAILWAY_TOKEN` lives only in GitHub's encrypted secrets, never in the repo.

## Bonus: live CLI access from your own machine

On your Mac (open internet), the CLI works directly — this is the `kalshi_bot`
pattern:

```bash
npm install -g @railway/cli
railway login                 # browser OAuth
railway link                  # pick the loop-engineering project
railway logs -n 50            # live service logs
railway volume files list /   # browse the volume
railway setup agent -y        # installs Railway's MCP server + skills for Claude Code
```

`railway setup agent -y` wires Railway's own MCP server into local Claude Code, so
a *locally-run* assistant gets live Railway tools. (A chat session in the web
sandbox still can't use it — the egress block applies to the MCP server's API calls
too — which is why the GitHub-Action sync above is the durable path here.)
