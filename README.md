# plant-monitor-deployment

GitOps deployment of [plant-monitor](https://github.com/owellnitz/plant-monitor)
to a LAN-only x86 Linux mini PC. Runs the prebuilt GHCR image plus its
dependencies (Mosquitto, Postgres); no build happens on the device.

```
CI on owellnitz/plant-monitor (main push)
   │  builds + pushes ghcr.io/owellnitz/plant-monitor/backend:latest
   ▼
GHCR (private — PAT pull)
   │  Watchtower polls every 5 min ──► recreates `backend` on new :latest
   ▼
Mini PC (static LAN IP, no inbound)        ◄── systemd timer: git pull && compose up -d
   mqtt :1883   db (internal)   backend :80
        ▲
   ESP32-C3 firmware publishes to mqtt://<static-ip>:1883
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

### 3. GitHub token (private repos)

Both this deploy repo and the GHCR backend image are **private**, so the mini
PC needs a token to pull them. One classic PAT covers both. Create one at
GitHub → Settings → Developer settings → **Personal access tokens (classic)**
with scopes:

- `repo` — clone/pull this private deploy repo
- `read:packages` — pull the private GHCR image (`repo` alone is **not** enough)

Call it `<TOKEN>` below.

### 4. Clone this repo and set the secret

```sh
sudo git clone https://<TOKEN>@github.com/owellnitz/plant-monitor-deployment.git /opt/plant-monitor-deployment
cd /opt/plant-monitor-deployment
sudo cp .env.example .env
sudo nano .env        # set POSTGRES_PASSWORD: openssl rand -base64 18 | tr '+/' '-_'
```

The token is stored in `/opt/plant-monitor-deployment/.git/config` (root-only),
so future `sudo git pull` authenticates automatically. To rotate it later:
`sudo git remote set-url origin https://<NEW_TOKEN>@github.com/owellnitz/plant-monitor-deployment.git`.
`.env` is gitignored — `git pull` never touches it.

### 5. Log in to GHCR (root)

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

### 6. Install the sync timer

```sh
sudo cp systemd/plant-monitor.service systemd/plant-monitor.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now plant-monitor.timer
```

First fire (within ~1 min) pulls the images and starts the stack. Verify:

```sh
sudo systemctl start plant-monitor.service   # force an immediate run
docker compose -f /opt/plant-monitor-deployment/compose.yml ps
```

The repo path is hardcoded as `/opt/plant-monitor-deployment` in
`plant-monitor.service` (`WorkingDirectory`). Clone elsewhere → edit that line.

### 7. Point the firmware at the broker

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
- **Web UI**: `http://<static-ip>`
- **Readings**: `docker compose exec db psql -U plantmonitor -c 'SELECT * FROM readings;'`

## Files

| Path | Purpose |
|------|---------|
| `compose.yml` | mqtt + db + backend (GHCR image) + watchtower; app on `:80` |
| `mosquitto/mosquitto.conf` | Broker config (anonymous, LAN-only) |
| `.env.example` | Template for `POSTGRES_PASSWORD` |
| `systemd/plant-monitor.{service,timer}` | Git-pull sync loop |
