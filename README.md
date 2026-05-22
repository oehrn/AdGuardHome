# AdGuard Home + Unbound — Docker Stack

Network-wide ads & trackers blocking DNS server (AdGuard Home) paired with
a local recursive resolver (Unbound), deployed via Docker Compose.

This repository ships a minimal stack designed to be deployed by a stack
manager (Komodo, Portainer, Dockge, etc.) or directly with `docker compose`.

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

## Point AdGuard Home at Unbound

Once the wizard is done, configure Unbound as the only upstream:

1. Open **Settings → DNS settings** in the AdGuard UI.
2. **Upstream DNS servers** — replace the defaults with just:
   ```
   unbound
   ```
   Docker's embedded DNS resolves the service name to the Unbound
   container's address on the internal `dnsnet` network (port 53).
3. **Bootstrap DNS servers** — keep `1.1.1.1` and `9.9.9.9`. These are
   only used to resolve hostnames inside DoH/DoT URLs, which Unbound
   does not do; in practice they are rarely contacted because Unbound
   talks to root servers by IP.
4. Click **Test upstreams** and confirm OK.
5. **DNSSEC enabled in AdGuard** — leave **off**. Unbound already
   validates DNSSEC; enabling validation in both layers is redundant
   and occasionally produces spurious SERVFAILs.

## DNS architecture

```
client  ──>  AdGuard Home  ──>  Unbound  ──>  root / TLD / authoritative
              (filter+UI)        (recursor)
```

- **AdGuard Home** handles filter lists, per-client rules, stats, and the
  admin UI. It does *not* recurse when paired with Unbound.
- **Unbound** is an iterative recursive resolver with local DNSSEC
  validation and root hints. No third-party resolver (Quad9, Cloudflare,
  Google, etc.) sees your queries by default.

Trade-offs:

| Pro | Con |
|---|---|
| No third-party telemetry on your DNS traffic | Cold cache is ~50–200 ms slower than CDN-backed resolvers |
| Local DNSSEC validation | More moving parts (two containers, two health surfaces) |
| Root hints stay fresh, no upstream lock-in | TLD servers rarely edge-cached → higher first-hit latency |

A DoT-forwarding fallback is shipped commented out at the bottom of
`unbound/unbound.conf`. Uncomment if root iteration becomes unreliable
(e.g. ISP blocks outbound port 53).

### Common pitfall — REFUSED loops

Never list your local router as an upstream DNS server in AdGuard. Most
consumer routers (and many internal DNS appliances) do not perform
recursion and will return `REFUSED`, which AdGuard logs noisily. Use the
router only as a **Private reverse DNS server** target for `*.arpa`
zones, or as a Conditional Forwarding entry for local zones — never as
an upstream.

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
weeks, since both images here use the `:latest` tag and breaking
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
docker logs unbound -f
docker exec adguardhome /opt/adguardhome/AdGuardHome -s status

# DNS smoke tests against the stack
dig @<host-ip> example.com
dig @<host-ip> +dnssec dnssec.works    # response should carry the AD flag
```

## License

MIT — see [LICENSE](LICENSE).
