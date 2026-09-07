# plant-monitor-deployment

GitOps deployment of [plant-monitor](https://github.com/owellnitz/plant-monitor)
to a LAN-only x86 Linux mini PC. Runs the prebuilt GHCR image plus its
dependencies (Mosquitto, Postgres); no build happens on the device.

> **Disclaimer:** Personal hobby project, published as-is for reference — no
> warranty, no support. The stack assumes a trusted home LAN with no inbound
> access: the MQTT broker allows anonymous connections and the web UI has no
> auth. Do not expose it to the internet without adding auth. Adapt IPs, names
> and paths to your own setup.

```
CI on owellnitz/plant-monitor (main push)
   │  builds + pushes ghcr.io/owellnitz/plant-monitor/backend:latest
   ▼
GHCR (private — PAT pull)
   │  Watchtower polls every 5 min ──► recreates `backend` on new :latest
   ▼
Mini PC (static LAN IP, no inbound)        ◄── systemd timer: git pull && compose up -d
   mqtt :1883   db (internal)   backend (internal)   caddy :80/:443
        ▲                                                 │
        │                          https://<PLANT_HOST> ──┤  browsers (PWA)
        │                     http://<static-ip>/api/firmware ──┘  ESP32 OTA
   ESP32-C3 publishes to mqtt://<static-ip>:1883
```

## Two update loops

| Loop | Watches | Mechanism | Cadence |
|------|---------|-----------|---------|
| Image | GHCR `:latest` digest | Watchtower, label-scoped to `backend` | 5 min |
| Config | this git repo | systemd `plant-monitor.timer` → `git pull && docker compose up -d --build` | 5 min |

Both are **pull-based** — the mini PC has no inbound access. Watchtower only
touches the labeled `backend` container; `db` and `mqtt` are never
auto-recreated.

## HTTPS

The frontend is a PWA. Service workers and the Web Push API require a **secure
context**, which `http://<static-ip>` is not. Caddy terminates TLS with a real
Let's Encrypt certificate while the mini PC stays LAN-only.

This works via the ACME **DNS-01** challenge: Let's Encrypt never connects to
the mini PC, it only reads a TXT record Caddy writes through the deSEC API. So
the hostname may resolve to a private address and **no port is forwarded**.

Two exceptions to "everything is HTTPS":

- `/api/firmware/*` stays reachable over plain HTTP on `<static-ip>`. The
  ESP32-C3 is `no_std` with no TLS stack and no DNS. Never add
  `UseHttpsRedirection`, `UseHsts` or `[RequireHttps]` to the backend — devices
  would keep publishing readings while reporting `"ota":"unreachable"` forever.
- Every other path on `<static-ip>` redirects to `https://<PLANT_HOST>`.

## One-time setup on the mini PC

### 1. Static IP

A **DHCP reservation** on the router (bind the NIC's MAC to an IP) is the
easiest option — and make sure the address is reserved even if it falls inside
the DHCP pool, or it can be handed to another device.

To set it on the host instead, Ubuntu uses **Netplan**. On WiFi the block needs
SSID + passphrase or the interface won't associate. Find the interface with
`ip -br link` (WiFi shows as `wl...`), then edit `/etc/netplan/*.yaml`:

```yaml
network:
  version: 2
  wifis:
    <wifi-iface>:
      dhcp4: false
      addresses: [192.168.1.50/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1]
      access-points:
        "YOUR_SSID":
          password: "YOUR_WIFI_PASSWORD"
```

Needs `wpasupplicant` (`sudo apt install -y wpasupplicant`). Apply with
`sudo netplan apply` (preview with `sudo netplan try`).

Note the address — called `<static-ip>` below. It must match the firmware's
`mqtt_host`.

### 2. Docker Engine

```sh
sudo apt update && sudo apt install -y git curl
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

### 3. GitHub token (GHCR image)

This repo is public, the GHCR backend image is **private**. Create a classic
PAT at GitHub → Settings → Developer settings → **Personal access tokens
(classic)** with the single scope `read:packages`. Called `<TOKEN>` below.

### 4. Hostname and token (deSEC)

1. Register a free domain at [desec.io](https://desec.io) → **dynDNS**, e.g.
   `plants.dedyn.io`. This is `<PLANT_HOST>`.
2. Add an `A` record — subname empty, value `<static-ip>`, TTL 3600.
3. **Token Management → +** → copy the secret, shown only once. This is
   `<DESEC_TOKEN>`. Caddy only writes `_acme-challenge` `TXT` records, so the
   token can be restricted to those via
   [RRset policies](https://desec.readthedocs.io/en/latest/auth/tokens.html).

**Do not run a dynDNS client against this domain.** deSEC hands out
`update.dedyn.io` credentials on signup; a router configured with them would
replace the `A` record with the public WAN address. The record is static.

The hostname appears in public Certificate Transparency logs. Informational
only — the address is unroutable from outside and nothing listens for external
connections.

### 5. Check that the name resolves on the LAN

A public name resolving to a private IP is also the signature of a DNS
rebinding attack, so some routers strip the answer. Check from a LAN client:

```sh
dig +short @ns1.desec.io <PLANT_HOST> A   # the record itself
dig +short               <PLANT_HOST> A   # the router's resolver
```

Both must return `<static-ip>`.

**Query the router only after the A record exists.** A lookup made beforehand
gets cached as a negative answer and keeps returning `NOERROR` with `ANSWER: 0`
for the TTL — indistinguishable from filtering. Wait out the TTL, or compare
against `dig +short @1.1.1.1 <PLANT_HOST> A` before concluding anything.

If the router really does filter, either configure it to hand clients a public
resolver over DHCP, or set DNS per device that opens the PWA — `1.1.1.1`, or
[NextDNS](https://nextdns.io) with `<PLANT_HOST>` allowlisted so rebind
protection stays on everywhere else.

The ESP32 is unaffected either way; `mqtt_host` is a literal IP.

### 6. Clone and set the secrets

```sh
sudo git clone https://github.com/owellnitz/plant-monitor-deployment.git /opt/plant-monitor-deployment
cd /opt/plant-monitor-deployment
sudo cp .env.example .env
sudo nano .env
```

| Variable | Value |
|----------|-------|
| `POSTGRES_PASSWORD` | `openssl rand -base64 18 \| tr '+/' '-_'` |
| `PLANT_HOST` | the deSEC hostname |
| `DESEC_TOKEN` | the deSEC token |
| `PLANT_IP` | `<static-ip>` — must equal the firmware's `mqtt_host` |

`.env` is gitignored — `git pull` never touches it. All four are mandatory:
`compose up` aborts if one is missing, so set them **before** the sync timer
pulls a commit that needs them.

### 7. Log in to GHCR (root)

The systemd service runs `docker compose` as **root**, so root must hold the
GHCR creds. Do this *before* the first `compose up`:

```sh
echo <TOKEN> | sudo docker login ghcr.io -u owellnitz --password-stdin
```

Writes `/root/.docker/config.json` — used by `compose up` to pull `backend`,
and read by Watchtower via the bind mount in `compose.yml`.

If login "succeeds" but pulls still 401, a credential helper hijacked the
creds. Check `sudo cat /root/.docker/config.json` for `credsStore`; if present,
remove that line and re-run the login so the auth blob is written inline.

### 8. Install the sync timer

```sh
sudo cp systemd/plant-monitor.service systemd/plant-monitor.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now plant-monitor.timer
```

First fire (within ~1 min) pulls the images, builds the Caddy image (~1 min,
cached afterwards) and starts the stack. Watch the certificate get issued with
`docker compose logs -f caddy`.

The repo path is hardcoded as `/opt/plant-monitor-deployment` in
`plant-monitor.service` (`WorkingDirectory`). Clone elsewhere → edit that line.

systemd reads units from `/etc/systemd/system`, not from the repo, so the
service reinstalls them itself when they differ from git and reloads systemd. A
pulled change to `systemd/` takes effect from the following run; the `cp` above
is only needed for this first install.

### 9. Point the firmware at the broker

In `firmware/config.toml` (in the main repo, gitignored):

| Key | Value |
|-----|-------|
| `mqtt_host` | `<static-ip>` |
| `mqtt_port` | `1883` |
| `backend_port` | `80` |

Rebuild + flash: `cargo run --release --features net`.

## Day-to-day

- **Ship app changes** → merge to `main` in `owellnitz/plant-monitor`. CI pushes
  `:latest`; Watchtower pulls it within 5 min. Nothing to do here.
- **Change the stack** → commit to this repo. The timer applies it within 5 min,
  or `sudo systemctl start plant-monitor.service` to apply now.
- **Web UI**: `https://<PLANT_HOST>`. After switching from `http://<static-ip>`
  the PWA must be reinstalled — a new origin means a new service worker and
  caches. On iOS: Safari → Share → Add to Home Screen.
- **Logs**: `docker compose logs -f backend` · `... -f caddy` for certificates ·
  `docker logs <watchtower-container>`
- **OTA health**: `mosquitto_sub -h <static-ip> -t 'sensors/#' -v` — the `ota`
  field should read `current`, `skipped` or `installed`, never `unreachable`.
- **Readings**: `docker compose exec db psql -U plantmonitor -c 'SELECT * FROM readings;'`

## Files

| Path | Purpose |
|------|---------|
| `compose.yml` | mqtt + db + backend (GHCR image) + caddy + watchtower |
| `caddy/Dockerfile` | Caddy rebuilt with the deSEC DNS module (for DNS-01) |
| `caddy/Caddyfile` | TLS termination, reverse proxy, plain-HTTP OTA carve-out |
| `mosquitto/mosquitto.conf` | Broker config (anonymous, LAN-only) |
| `.env.example` | Template for the four required variables |
| `systemd/plant-monitor.{service,timer}` | Git-pull sync loop |
