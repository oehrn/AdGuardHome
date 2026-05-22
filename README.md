# AdGuard Home — Docker Stack

Network-wide ads & trackers blocking DNS server (AdGuard Home), deployed via
Docker Compose.

This repository ships a minimal stack designed to be deployed by a stack
manager (Komodo, Portainer, Dockge, etc.) or directly with `docker compose`.

Upstream resolution is handled by **Quad9 DoT** (encrypted DNS over TLS,
no third-party logging, EU/CH-hosted). A previous version of this stack
shipped Unbound as a recursive sidecar; that was removed in favor of a
simpler single-container deployment — see commit history if you need the
Unbound-based variant.

## Requirements

- Docker Engine 24+ with Compose v2
- Linux host (host network mode is not supported on Docker Desktop)
- Ports `53/tcp`, `53/udp`, `80/tcp`, `3000/tcp` free on the host
- A bind-mounted or named volume for AdGuard Home's data and config

If port 53 is occupied (typical on Debian/Ubuntu hosts running
`systemd-resolved`):

```bash
ss -tlnp | grep :53
sudo systemctl disable --now systemd-resolved
```

You may also need to point `/etc/resolv.conf` at an external resolver
before the stack is healthy; see the AdGuard Home wiki entry on
[`resolved`](https://github.com/AdguardTeam/AdGuardHome/wiki/Docker#resolved-daemon).

## Configuration model

Unlike Pi-Hole's `FTLCONF_*` environment variables, AdGuard Home is
configured **entirely** through its `AdGuardHome.yaml` file, which is
created on first start via the setup wizard at `:3000`. The `.env` file
in this repository only carries:

- `TZ` — container timezone
- `ADGUARD_WEB_PORT` — optional host-side port override for the admin UI

Everything else (upstreams, DNSSEC, filter lists, allowed clients,
conditional forwarding, admin password) is configured in the web UI or by
editing the YAML directly while the container is stopped.

## Quick Start

```bash
git clone https://github.com/oehrn/AdGuardHome.git
cd AdGuardHome
cp .env.example .env
# Edit .env (set TZ)
docker compose up -d
```

## Initial Setup Wizard

1. Open `http://<host-ip>:3000` in a browser.
2. Set the admin username and password.
3. **Admin web interface:** listen on `0.0.0.0`, port `80`.
4. **DNS server:** listen on `0.0.0.0`, port `53`.
5. Finish the wizard. The UI is now reachable at `http://<host-ip>/`.

## Configure upstream DNS

After the wizard, open **Settings → DNS settings** and paste the
following into **Upstream DNS servers** (adjust the local-zone forwards
to your environment — the example below routes local zones to a UDM SE
at `10.5.9.1`):

```
[/mtrolle.com/]10.5.9.1
[/lokal/]10.5.9.1
[/vpn/]10.5.9.1
[/lps/]10.5.9.1
tls://dns.quad9.net
tls://dns.quad9.net:853#149.112.112.112
```

How the `[/zone/]server` prefix works: AdGuard Home routes any query for
a hostname inside `*.mtrolle.com`, `*.lokal`, `*.vpn`, or `*.lps`
directly to the local router (`10.5.9.1`). Everything else is sent to
Quad9 over DoT. This replaces Pi-Hole's "Conditional Forwarding"
feature.

The second Quad9 line uses the explicit-IP form
(`tls://dns.quad9.net:853#149.112.112.112`) so the resolver can reach
the secondary Quad9 address even when the primary is unreachable.

**Bootstrap DNS servers:**

```
9.9.9.9
149.112.112.112
```

Bootstrap is only used to resolve the DoT hostname (`dns.quad9.net`)
once at startup.

**DNSSEC** — leave enabled. Quad9 validates upstream and sets the AD
flag; AdGuard Home re-validates on the way back to the client. The two
layers are compatible over DoT (no spurious SERVFAILs).

Click **Test upstreams** and confirm OK.

## DNS architecture

```
client  ──>  AdGuard Home  ──>  Quad9 (DoT 853)
              (filter+UI)        (recursor+DNSSEC)
```

- **AdGuard Home** handles filter lists, per-client rules, stats, DNS
  rewrites (split-horizon), and the admin UI.
- **Quad9** handles recursion and DNSSEC validation upstream. Queries
  leave the LAN encrypted (DoT, port 853).

Trade-offs vs. self-hosted recursion (e.g. Unbound):

| Pro | Con |
|---|---|
| Single container, far less operational surface | Quad9 sees the query stream (no-logging policy, but not zero trust) |
| Fast cold cache (Quad9 anycast + edge caches) | No local recursion — you depend on Quad9 availability |
| DoT encrypts queries over the WAN | TLS adds a few ms per cold query |
| No root-hints / DNSSEC trust-anchor maintenance | |

### Common pitfall — REFUSED loops

Never list your local router as a *general* upstream DNS server in
AdGuard. Most consumer routers (and many internal DNS appliances) do
not perform recursion and will return `REFUSED`. Use the router only
via the `[/zone/]server` prefix shown above (so it only sees queries
for local zones), as a **Private reverse DNS server** target for
`*.arpa` zones, or as a Conditional Forwarding entry for local zones —
never as a catch-all upstream.

## Deploying with a stack manager

### Komodo

1. Create a new Stack.
2. Set the linked repository to this repo, branch `main`.
3. Leave `file_paths` empty (Compose lives at repo root).
4. Open the Environment tab and paste the contents of `.env.example`,
   adjusting `TZ` (and optionally `ADGUARD_WEB_PORT`).
5. Deploy.

Auto-update via Komodo's `Global Auto Update` procedure is **opt-in** —
enable it only after the stack has run stable for at least one to two
weeks, since the image here uses the `:latest` tag and breaking
upstream changes can land without warning.

### Portainer / Dockge

1. Add stack from Git repository.
2. Repository URL: `https://github.com/oehrn/AdGuardHome`
3. Compose path: `docker-compose.yml`
4. Add environment variables (or upload `.env`).
5. Deploy.

## Updating

```bash
docker compose pull
docker compose up -d
```

Or trigger a pull + redeploy from your stack manager.

To roll back after a bad update, pin a specific tag in
`docker-compose.yml`, e.g. `adguard/adguardhome:v0.107.58`, and redeploy.

## Wizard reset

If you need to re-run the initial setup wizard (for example after
losing the admin password), remove the generated config and restart:

```bash
docker exec adguardhome rm /opt/adguardhome/conf/AdGuardHome.yaml
docker compose restart adguardhome
```

## Useful commands

```bash
docker compose ps
docker logs adguardhome -f
docker exec adguardhome /opt/adguardhome/AdGuardHome -s status

# DNS smoke tests against the stack
dig @<host-ip> example.com
dig @<host-ip> +dnssec dnssec.works    # response should carry the AD flag
```

## License

MIT — see [LICENSE](LICENSE).
