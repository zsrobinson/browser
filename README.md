# browser

A signed-in Chromium on the Synology that Claude drives as a connector, and
that you can watch (or take over) at **https://browser.zsrobinson.com**.

```
          phone / laptop                        claude.ai (any device, scheduled tasks)
                │                                          │
    browser.zsrobinson.com                    browsermcp.zsrobinson.com
    (Access: you only)                        (Access + Managed OAuth)
                │                                          │
                └────────────── Cloudflare Tunnel ─────────┘
                                      │  (no ports open on the NAS)
 ┌──────────────────────── NAS · docker compose ─────────────────────────┐
 │  cloudflared ──► browser container                                     │
 │                   ├─ :3000  Selkies web viewer ─┐                      │
 │                   ├─ :8931  Playwright MCP ─────┼─ localhost CDP ─► Chromium
 │                   └─ ./data  Chromium profile (your signed-in sessions) │
 └────────────────────────────────────────────────────────────────────────┘
```

| Container     | Job                                                         | RAM cap |
|---------------|-------------------------------------------------------------|---------|
| `browser`     | Chromium, the web viewer, and the MCP server Claude calls   | 1400 MB |
| `cloudflared` | The only way in or out                                      | 96 MB   |

`browser` is linuxserver's Chromium image with Playwright MCP added as a
second s6-supervised service (`image/`). Both live in one container so the
MCP server reaches Chromium over localhost — no proxy, no extra network.

## How logins work

You sign in, never Claude. When Claude hits a login page it sends you the
viewer link and stops. You open https://browser.zsrobinson.com (laptop is
easiest; phone works), sign in with Bitwarden as usual, and reply "done".
The profile keeps that session until the site expires it.

**Adding a new site needs no code:** sign in once in the viewer and it's
available to Claude.

## Deploy

1. **Check headroom** (2 GB box): `docker stats --no-stream` — want existing
   stacks under ~600 MB combined.
2. Clone onto the NAS, e.g. `/volume1/docker/browser`. Keep it **out of any
   shared folder** that syncs or is browsable — `data/` holds live session
   cookies.
3. `cp .env.example .env`, fill in `PUID`/`PGID` (run `id`) and the tunnel token.
4. Cloudflare: tunnel, two routes, two Access apps — see
   [docs/cloudflare.md](docs/cloudflare.md). **Do the Access apps before
   step 5**; the viewer has no login of its own.
5. `docker compose up -d --build` (first build takes a few minutes).
6. Open https://browser.zsrobinson.com → you should see Chromium. Sign in to AMC.
7. claude.ai → Settings → Connectors → Add custom connector →
   `https://browsermcp.zsrobinson.com/mcp`. **Name it "Home Browser"** —
   the skills refer to it by that name.
8. Save the two skills in `skills/` (zip each folder and upload under
   Settings → Capabilities → Skills, or save from Claude's skill card).
9. Test from your phone: *"what Dolby showtimes are at Columbia this Friday?"*

## Updating

`docker compose build --pull && docker compose up -d` picks up new Chromium
and Playwright MCP versions. Once everything works, pin `@playwright/mcp@<version>`
in `image/Dockerfile` so updates are deliberate.

## Speed

- Playwright MCP reads pages as an accessibility tree (text), not
  screenshots — the fast path for browser agents.
- For browsing tasks, pick a faster model in the claude.ai model picker.
- The big win is phase 2 below.

## Phase 2 (once the AMC flow is known): fast tools

Every browser step is a model round trip. A booking done step-by-step is
~20–30 calls; with purpose-built tools it's 2–3. After a few real bookings,
add tools like `find_showtimes(date, theaters, formats)` and
`reserve(showtime_id, seats)` as deterministic Playwright scripts against the
same Chromium (another s6 service in `image/`, or a small sidecar). Claude
then only decides; code executes. Same pattern for any site you use weekly.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Chromium crashes on heavy pages | confirm `shm_size: 1gb`; check `docker stats` against the 1400m cap |
| Chromium window closed in the viewer | `RESTART_APP=true` relaunches it within seconds |
| Claude's tools fail to reach the browser | `docker compose exec browser curl -s http://127.0.0.1:9222/json/version` should print Chromium's version JSON. If not, check `CHROME_CLI` contains `--remote-debugging-port=9222` and is **not** quoted. |
| MCP service not running | `docker compose logs browser \| grep -i mcp`; s6 restarts it automatically |
| claude.ai says "Authorization with the MCP server failed" | see the OAuth notes in docs/cloudflare.md |
| Build fails pulling the base image (`toomanyrequests`, timeouts) | ghcr.io may be down — check https://status.linuxserver.io. Set `BASE_IMAGE=linuxserver/chromium:latest` in `.env` to use the Docker Hub copy, then rebuild. If Docker Hub rate-limits, `sudo docker login` or wait an hour. |
| `path "./image" not found` | the project folder isn't the cloned repo — build from the folder that contains `image/` |

## Security notes

- Whoever can reach the viewer or the MCP endpoint controls every signed-in
  account. Both sit behind Cloudflare Access; never publish ports.
- Sign in only to sites you're comfortable with Claude operating. Keep banking
  and primary email out of this profile.
- Pages are untrusted input. Claude treats page text as data and confirms
  before any submit, reservation, purchase, or message.
- To revoke a site: sign out of it in the viewer.
