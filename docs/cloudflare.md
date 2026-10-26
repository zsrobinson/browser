# Cloudflare setup

Everything here is in the Cloudflare Zero Trust dashboard. Menu names move
around; the concepts don't.

## 1. Tunnel

Networks → Tunnels → Create tunnel (type: cloudflared). Copy the token into
`.env` as `TUNNEL_TOKEN`. Add two **public hostnames**:

| Hostname                     | Service              |
|------------------------------|----------------------|
| `browser.zsrobinson.com`     | `http://browser:3000`|
| `browsermcp.zsrobinson.com` | `http://browser:8931`|

Leave "HTTP Host Header" unset on the MCP route — Playwright MCP checks the
Host header against `--allowed-hosts`, which expects `browsermcp.zsrobinson.com`.

## 2. Access app: the viewer (`browser.zsrobinson.com`)

Access → Applications → Add → Self-hosted.

- Domain: `browser.zsrobinson.com`
- Policy: Allow → Emails → your email only
- Login method: one-time PIN (works anywhere) and/or Google
- Session duration: whatever you like (24h is reasonable)

The linuxserver image has no auth of its own; this app *is* the lock. For a
second layer you can also set `CUSTOM_USER` / `PASSWORD` env vars on the
`browser` service (basic auth).

## 3. Access app: the MCP endpoint (`browsermcp.zsrobinson.com`)

claude.ai custom connectors authenticate with **OAuth only** — they can't send
a static token. Cloudflare Access can act as the OAuth server so the MCP
container needs no auth code:

Access → Applications → Add → Self-hosted.

- Domain: `browsermcp.zsrobinson.com`
- Policy: same as above (your email only)
- Advanced / OAuth settings: enable **Managed OAuth** (dynamic client registration)
- Allowed redirect URIs:
  - `https://claude.ai/api/mcp/auth_callback`
  - `https://claude.com/api/mcp/auth_callback`

Check: `curl -s https://browsermcp.zsrobinson.com/.well-known/oauth-authorization-server`
should return JSON with `authorization_endpoint`, `token_endpoint`, and
`registration_endpoint`.

Then in claude.ai: Settings → Connectors → Add custom connector →
`https://browsermcp.zsrobinson.com/mcp` → Connect → sign in with your email.

### If claude.ai's connect step fails

People have reported "Authorization with the MCP server failed" on claude.ai
with OAuth setups that work in Claude Code. Two things to try:

1. **MCP server portal** (Access → AI controls → MCP servers / portals):
   register `https://browsermcp.zsrobinson.com/mcp` as a server, put a portal
   on its own hostname with Managed OAuth enabled and the same two redirect
   URIs, and add the *portal* URL as the connector.
2. Re-check the redirect URIs and that the `.well-known` endpoints above
   respond without an Access login page in the way.

Reference: https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/secure-mcp-servers/

**Never** "fix" connector auth by removing the Access app — an open MCP
endpoint is a remote control for every account you're signed in to.

## 4. TURN key for a smoother viewer (optional)

The viewer streams over WebSockets through the tunnel by default, which
stalls whenever the connection is congested. WebRTC mode sends video over UDP
via a TURN relay instead; Cloudflare's is free for this kind of use.

1. Dashboard → **Realtime** (formerly "Calls") → **TURN Server** → Create.
   Name it `browser-viewer`.
2. Copy the **Turn Token ID** and the **API Token** (shown once).
3. In `.env`:
   ```
   VIEWER_MODE=webrtc
   CLOUDFLARE_TURN=true
   CLOUDFLARE_TURN_KEY_ID=<Turn Token ID>
   CLOUDFLARE_TURN_API_TOKEN=<API Token>
   ```
4. `sudo docker compose up -d` (no rebuild).

Signaling still goes through the tunnel and Access, so the login is unchanged.
To go back, set `VIEWER_MODE=websockets` (or remove the lines) and `up -d`.
The viewer sidebar can also switch modes live.
