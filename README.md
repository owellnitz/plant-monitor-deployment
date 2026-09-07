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
| Config | this git repo | systemd `plant-monitor.timer` → `git pull && docker compose up -d` | 5 min |

The mini PC has no inbound access, so both loops are **pull-based**. Watchtower
restarts the app on a new image; the git-pull timer applies changes to
`compose.yml` / `mosquitto.conf`. Watchtower only touches the labeled
`backend` container — `db` and `mqtt` are never auto-recreated.

## One-time setup on the mini PC

### 1. Static IP

Give the mini PC a fixed LAN address so the firmware's hardcoded
`mqtt_host` keeps working. Easiest and distro-agnostic is a **DHCP
reservation** on your router (bind the NIC's MAC to an IP) — recommended.

To set it on the host instead, Ubuntu manages the NIC with **Netplan**. The
mini PC is on WiFi, so use a `wifis:` block — it must carry the SSID +
passphrase, otherwise the interface won't associate. Find the interface name
with `ip -br link` (WiFi NICs show as `wl...`), then edit the Netplan file
under `/etc/netplan/` (e.g. `/etc/netplan/01-netcfg.yaml`):

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

Netplan drives WiFi via `wpa_supplicant` — make sure it's installed
(`sudo apt install -y wpasupplicant`). Apply with `sudo netplan apply`
(preview first with `sudo netplan try`).

NetworkManager — only on Ubuntu Desktop, if you prefer `nmcli`:

```sh
nmcli con mod "<con>" ipv4.method manual \
  ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns 192.168.1.1
nmcli con up "<con>"
```

Pick an address outside the router's DHCP pool. Note it — call it `<static-ip>`.

### 2. Docker Engine

Minimal Ubuntu may lack `git`/`curl` — install them first:

```sh
sudo apt update && sudo apt install -y git curl
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

### 3. GitHub token (GHCR image)

This deploy repo is public, but the GHCR backend image is **private**, so the
mini PC needs a token to pull it. Create one at
GitHub → Settings → Developer settings → **Personal access tokens (classic)**
with the single scope:

- `read:packages` — pull the private GHCR image

Call it `<TOKEN>` below.

### 4. HTTPS hostname (deSEC)

The frontend is a PWA. Service workers and the Web Push API only run in a
**secure context**, which `http://<static-ip>` is not — so without TLS there
are no push notifications. Caddy solves this with a real Let's Encrypt
certificate while the mini PC stays LAN-only.

That works because of the **DNS-01** challenge: Let's Encrypt never connects to
the mini PC, it only reads a TXT record that Caddy writes via the deSEC API. So
the hostname may resolve to a private address and no port is forwarded.

1. Register a free domain at [desec.io](https://desec.io) → **dynDNS**, e.g.
   `plants.dedyn.io`. Call it `<PLANT_HOST>`.
2. Set its `A` record to the mini PC's `<static-ip>` (e.g. `192.168.1.50`).
3. Create a token under **Token management**, scoped to that domain. Call it
   `<DESEC_TOKEN>`. Caddy only ever writes `_acme-challenge` `TXT` records, so
   the token can be restricted to those.

**Do not run a dynDNS client against this domain.** deSEC hands out
`update.dedyn.io` credentials on signup, and a router configured with them
would replace the `A` record with the *public* WAN address. The record here is
static and must keep pointing at the private `<static-ip>`; Caddy never touches
it.

**Router: DNS rebind protection.** Many routers drop public DNS answers that
resolve to a private IP, so `<PLANT_HOST>` will not resolve on the LAN until
that is dealt with. Verify from a LAN client:

```sh
dig +short @ns1.desec.io <PLANT_HOST> A   # the record itself
dig +short @1.1.1.1      <PLANT_HOST> A   # a resolver that does not filter
dig +short               <PLANT_HOST> A   # the router's resolver
```

The record is fine if the first two answer with `<static-ip>`. If only the
third is empty — typically `status: NOERROR` with `ANSWER: 0` — the router is
stripping the private address.

- **FRITZ!Box**: whitelist the name under *Heimnetz → Netzwerk →
  Netzwerkeinstellungen → DNS-Rebind-Schutz*.
- **Vodafone Station**: filters, and exposes no setting for it. Either hand
  clients a public resolver over DHCP if the firmware allows it, or set DNS
  (e.g. `1.1.1.1`) manually per device in its Wi-Fi settings.

### 5. Clone this repo and set the secrets

```sh
sudo git clone https://github.com/owellnitz/plant-monitor-deployment.git /opt/plant-monitor-deployment
cd /opt/plant-monitor-deployment
sudo cp .env.example .env
sudo nano .env
```

| Variable | Value |
|----------|-------|
| `POSTGRES_PASSWORD` | `openssl rand -base64 18 \| tr '+/' '-_'` |
| `PLANT_HOST` | the deSEC hostname from step 4 |
| `DESEC_TOKEN` | the deSEC token from step 4 |
| `PLANT_IP` | `<static-ip>` from step 1 — must equal the firmware's `mqtt_host` |

`.env` is gitignored — `git pull` never touches it. All four are mandatory:
`compose up` aborts if one is missing, so set them **before** the sync timer
pulls a commit that needs them.

### 6. Log in to GHCR (root)

The systemd service runs `docker compose` as **root**, so root must hold the
GHCR creds. Do this *before* the first `compose up`:

```sh
echo <TOKEN> | sudo docker login ghcr.io -u owellnitz --password-stdin
```

Writes `/root/.docker/config.json` — which `compose up` uses to pull `backend`,
and which Watchtower reads via the bind mount in `compose.yml` to pull updates.

If login "succeeds" but pulls still 401, a credential helper hijacked the
creds. Check `sudo cat /root/.docker/config.json` for `credsStore`; if present,
remove that line and re-run the login so the auth blob is written inline.

### 7. Install the sync timer

```sh
sudo cp systemd/plant-monitor.service systemd/plant-monitor.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now plant-monitor.timer
```

First fire (within ~1 min) pulls the images, builds the Caddy image (~1 min,
cached afterwards) and starts the stack. Verify:

```sh
sudo systemctl start plant-monitor.service   # force an immediate run
docker compose -f /opt/plant-monitor-deployment/compose.yml ps
```

The repo path is hardcoded as `/opt/plant-monitor-deployment` in
`plant-monitor.service` (`WorkingDirectory`). Clone elsewhere → edit that line.

**The units are copies, not symlinks.** The sync loop applies `compose.yml` and
`mosquitto.conf` from git, but it cannot update itself: a change to
`systemd/plant-monitor.{service,timer}` in this repo only takes effect after
re-running the `cp` and `daemon-reload` above. Worth checking after any pull
that touched `systemd/`.

### 8. Point the firmware at the broker

In `firmware/config.toml` (in the main repo, gitignored):

| Key | Value |
|-----|-------|
| `mqtt_host` | `<static-ip>` |
| `mqtt_port` | `1883` |

Rebuild + flash: `cargo run --release --features net`.

## Day-to-day

- **Ship app changes** → merge to `main` in `owellnitz/plant-monitor`. CI
  pushes `:latest`; Watchtower pulls it within 5 min. Nothing to do here.
- **Change the stack** (ports, broker config, add a service) → commit to this
  repo. The timer applies it within 5 min, or run
  `sudo systemctl start plant-monitor.service` to apply now.
- **Logs**: `docker compose logs -f backend`
- **Watchtower activity**: `docker logs <watchtower-container>`
- **Certificate issuance / renewal**: `docker compose logs -f caddy`
- **Web UI**: `https://<PLANT_HOST>` — use this, not the IP. `http://<static-ip>`
  redirects there, and only a secure origin gets a service worker and push.
- **OTA**: the ESP32 has no TLS stack, so `/api/firmware/*` stays reachable on
  plain `http://<static-ip>` — that one path is deliberately not redirected.
  Devices need no reprovisioning. Check with:
  `curl http://<static-ip>/api/firmware/latest?current=firmware-v0.0.0`
- **Readings**: `docker compose exec db psql -U plantmonitor -c 'SELECT * FROM readings;'`

## Files

| Path | Purpose |
|------|---------|
| `compose.yml` | mqtt + db + backend (GHCR image) + caddy + watchtower; app on `:443` |
| `caddy/Dockerfile` | Caddy rebuilt with the deSEC DNS module (for DNS-01) |
| `caddy/Caddyfile` | TLS termination + reverse proxy to `backend:8080` |
| `mosquitto/mosquitto.conf` | Broker config (anonymous, LAN-only) |
| `.env.example` | Template for `POSTGRES_PASSWORD`, `PLANT_HOST`, `DESEC_TOKEN` |
| `systemd/plant-monitor.{service,timer}` | Git-pull sync loop |
