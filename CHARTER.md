# Agentic Monitoring Stack — Setup Charter

This is a self-contained setup charter for a home-network monitoring and management stack on a dedicated Linux box. Reference build: Raspberry Pi 5 + USB SSD. Equally valid: an x86 mini PC, an old desktop or laptop, a VM, or any bare-metal machine. Debian-family (Debian/Ubuntu/Raspberry Pi OS) is assumed throughout; adapt package commands for other distros. It is written to be handed directly to a Claude Code instance running on your human's machine. It assumes zero prior context beyond what's written here.

---

## 1. Charter — read this first

You (the Claude Code agent reading this) are setting up a centralized home-network monitoring and management stack on the dedicated box your human has chosen (see Section 2, Question 0). This design is distilled from three working production deployments — it is proven, not experimental. Your job is to **adapt it to the local network**, not to reinvent it. Do not substitute alternative tools for the stack listed below unless your human explicitly asks for a swap.

Rules for how you work through this document:

- **Work phase by phase, in order.** Each phase has prerequisites that later phases depend on (e.g., Docker's data-root must be relocated before you pull any images).
- **Each phase ends with a "### Verify" block.** Do not advance to the next phase until every command in that block passes and produces the expected output. If a check fails, diagnose before retrying — don't paper over it by skipping ahead.
- **Report progress to your human between phases.** A short status update after each phase (what was done, what verified, anything that needs a decision) is expected — this is a multi-hour project spread across a session or several.
- **Never expose any dashboard or service in this stack to the public internet.** Everything here is LAN-only or Tailscale-only. No port forwarding on the router, no Cloudflare Tunnel, no reverse proxy with a public hostname. If your human asks for remote access later, that's a deliberate, separate decision — flag it, don't just do it.
- **Ask before destructive actions.** Formatting the SSD, changing router DHCP settings, and disabling the host's default DNS all need a human "yes" before you touch them — see Section 2.
- This document contains placeholders in `<ANGLE-BRACKET-CAPS>` form. Resolve every one of them with your human before you start Phase 0; do not guess network details.

---

## 2. Ask your human first

Before touching anything, get answers to these. Write them down somewhere durable (a scratch note, a README in the stack directory) so you don't have to re-ask mid-build.

| # | Question | Why it matters |
|---|---|---|
| 0 | What hardware is this stack going on — a Raspberry Pi, another physical Linux box, or a VM? And what data storage does it have? | Selects the Phase 0 branch (0A Pi / 0B generic); VMs skip UPS-USB and UAS concerns; a machine with one big disk skips the data-root relocation. |
| 1 | What static IP should the Pi have, and what's the LAN subnet (e.g. `192.168.1.0/24`)? | Every service in this stack is reached by a fixed IP or hostname. A DHCP-assigned Pi will break DNS records and dashboard links when it changes. |
| 2 | Can the router's DHCP settings be changed to hand out the Pi as the DNS server? Does your human have router admin access? | Pi-hole only ad-blocks/resolves for clients that are told to use it. Without router access you can still run Pi-hole, but only devices manually configured to use it will benefit. |
| 3 | Is Tailscale already set up on this account, and is your human willing to enable Tailscale SSH? | Tailscale SSH is used as the emergency access path independent of the LAN's DNS/router state. |
| 4 | What hostname should the Pi have? | Used throughout local DNS records and dashboard links. |
| 5 | Which other machines (if any) should get MeshCentral agents, and what OS are they (Windows/macOS/Linux)? | Determines which server-baked installers you'll generate in Phase 5. |
| 6 | What phone OS (iOS/Android) will receive ntfy push alerts? | Just for pointing your human at the right app store listing — no technical difference in setup. |
| 7 | Confirm: is the data drive (if any) blank/formattable, or does it have existing data your human needs preserved? | Phase 0 partitions and formats it. This is destructive — get an explicit yes. |
| 8 | Are any of the machines work-issued or employer-owned? | **Work machines get NO agents of any kind — no MeshCentral, no Telegraf, no remote management, nothing installed.** The only permissible monitoring is a passive reachability check (ping/TCP) from Uptime Kuma, and only if your human wants it. This is a hard boundary: employer hardware and personal infrastructure must not share tooling, credentials, or management planes. |

Fill in the placeholder table below once you have answers:

| Placeholder | Meaning | Your human's answer |
|---|---|---|
| `<PI-STATIC-IP>` | Static LAN IP for the Pi | |
| `<GATEWAY-IP>` | LAN gateway (router) IP | |
| `<LAN-SUBNET>` | LAN subnet, e.g. `192.168.1.0/24` | |
| `<PI-HOSTNAME>` | Hostname for the Pi | |
| `<TAILNET-NAME>` | Tailscale tailnet name (from the admin console) | |
| `<SSD-DEVICE>` | Device path for the SSD, e.g. `/dev/sda` (confirm with `lsblk` — do not assume) | |
| `<SSD-UUID>` | Filled in after formatting in Phase 0 | |
| `<STACK-DIR>` | Directory tree root on the SSD, e.g. `/srv/stack` | |
| `<PIHOLE-WEBPASS>` | Pi-hole web admin password (generate, store safely — not the same as the API key) | |
| `<INFLUX-ADMIN-PASS>` | InfluxDB admin UI password | |
| `<INFLUX-ADMIN-TOKEN>` | InfluxDB admin token (generated in Phase 3) | |
| `<INFLUX-WRITE-TOKEN>` | InfluxDB write-only token for Telegraf agents (generated in Phase 3) | |
| `<GRAFANA-ADMIN-PASS>` | Grafana admin password | |
| `<NTFY-ADMIN-PASS>` | ntfy admin user password | |
| `<NTFY-PUBLISHER-TOKEN>` | ntfy scoped write-only token (generated in Phase 4) | |
| `<TOPIC>` | ntfy topic name, e.g. `alerts` | |
| `<MESHCENTRAL-CERT-HOST>` | Stable address baked into MeshCentral's cert (Tailscale MagicDNS name recommended) | |
| `<PIHOLE-API-KEY>` | Pi-hole app/API key for dashboard widgets (Pi-hole UI → Settings → API — NOT the web password) | |
| `<KUMA-STATUS-PAGE-SLUG>` | Uptime Kuma status-page slug (created in Kuma's UI, used by the Homepage widget) | |

Placeholder names say PI for historical reasons; they mean the stack box, whatever it is.

---

## 3. Phase 0 — Base OS + storage

**Goal:** a booted, updated 64-bit Linux install with the data storage mounted (if applicable) and Docker's data-root relocated to it — all before any Docker image is pulled. Pick the branch that matches your human's answer to Section 2, Question 0.

### 3.A Raspberry Pi path

Use this branch for a physical Raspberry Pi 5.

#### 3.A.1 OS

- Flash **Raspberry Pi OS Lite, 64-bit** to the boot medium (SD card or the SSD itself if your human wants to boot from SSD — recommended if the Pi 5's bootloader supports USB boot, which it does by default on recent firmware; confirm with `rpi-eeprom-update`).
- First boot, then:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl jq gnupg lsb-release ca-certificates
sudo reboot
```

- Set the hostname to `<PI-HOSTNAME>`:

```bash
sudo raspi-config nonint do_hostname <PI-HOSTNAME>
```

- Set a static IP. If using NetworkManager (default on recent Pi OS):

```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses <PI-STATIC-IP>/24 ipv4.gateway <GATEWAY-IP> ipv4.dns "1.1.1.1" ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

#### 3.A.2 SSD

> [!warning] USB-SATA/NVMe bridge UAS quirk
> Some USB-to-SATA/NVMe bridge chips hang under load when using the USB Attached SCSI (UAS) protocol — symptoms are `uas_eh_abort`/`device_reset` messages in `dmesg` and the SSD dropping out under write pressure (exactly what a Docker data-root will do to it). This was seen with a JMicron bridge on a Pi 4B; the Pi 5's different USB controller and a different bridge chip may be totally fine with UAS — **UAS is faster, so only quirk it if it actually misbehaves.** If you see hangs, find the bridge's vendor:product ID with `lsusb`, then add `usb-storage.quirks=<VID>:<PID>:u` to `/boot/firmware/cmdline.txt` (same line, space-separated, no newline) and reboot.

- Identify the SSD (confirm before formatting — do not guess):

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

- Partition and format as ext4 (destructive — confirmed with human per Section 2):

```bash
sudo parted <SSD-DEVICE> --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L stackdata <SSD-DEVICE>1
```

- Get the UUID and add a `nofail` fstab entry (nofail so a missing/failed SSD doesn't hang the boot):

```bash
sudo blkid <SSD-DEVICE>1
```

Add to `/etc/fstab`:

```
UUID=<SSD-UUID>  /mnt/ssd  ext4  defaults,nofail  0  2
```

```bash
sudo mkdir -p /mnt/ssd
sudo mount -a
df -h /mnt/ssd
```

#### 3.A.3 Stack directory tree

```bash
sudo mkdir -p /mnt/ssd/<STACK-DIR>/{pihole/etc-pihole,pihole/etc-dnsmasq.d,influxdb/data,influxdb/config,telegraf,grafana/data,grafana/provisioning,ntfy/data,ntfy/config,kuma/data,meshcentral/data,homepage/config,homarr/appdata}
sudo ln -s /mnt/ssd/<STACK-DIR> ~/stack
```

(If you preferred `/srv/stack` directly, symlink accordingly — the rest of this document uses `~/stack` as shorthand for wherever `<STACK-DIR>` resolves.)

### 3.B Generic / VM path

Use this branch for any other physical Linux box (mini PC, old desktop/laptop) or a VM.

- Install **Debian or Ubuntu Server** (minimal install), 64-bit.
- Set the hostname:

```bash
sudo hostnamectl set-hostname <PI-HOSTNAME>
```

- Set a static IP via the distro's tool (`nmcli` on NetworkManager systems, or edit `/etc/netplan/*.yaml` on netplan systems), or configure a DHCP reservation on the router instead if that's simpler.
- Data storage: if there's a second disk, mount it by UUID with `nofail` in `/etc/fstab`, same pattern as the Pi path (3.A.2). If there isn't a second disk, a plain directory on the main disk is fine — the SD-wear rationale behind the Pi path's SSD relocation doesn't apply to real SSDs or virtual disks.
- Docker data-root relocation (4.1) is **optional** here — only do it if the OS disk is small.
- **VMs:** give the guest 4 GB+ RAM and 40 GB+ disk. All of this stack's services run fine virtualized except USB-UPS monitoring (7.4), which belongs on the hypervisor host, not the guest.
- Create the stack directory tree (adjust the base path if you're not using `/mnt/ssd`):

```bash
sudo mkdir -p <DATA-ROOT>/<STACK-DIR>/{pihole/etc-pihole,pihole/etc-dnsmasq.d,influxdb/data,influxdb/config,telegraf,grafana/data,grafana/provisioning,ntfy/data,ntfy/config,kuma/data,meshcentral/data,homepage/config,homarr/appdata}
sudo ln -s <DATA-ROOT>/<STACK-DIR> ~/stack
```

### Verify

```bash
df -h <DATA-ROOT>                   # data directory mounted (if a separate disk), correct size shown
cat /etc/fstab | grep stackdata     # entry present with nofail (if a separate disk is in use)
dmesg | grep -i uas                 # Pi/USB-SSD path only — empty or no abort/reset errors; apply the quirk above and re-verify if present
ls ~/stack                          # directory tree exists
```

Do not proceed to Phase 1 until the data directory survives a reboot cleanly (`sudo reboot`, then re-run `df -h`).

---

## 4. Phase 1 — Docker + Tailscale

### 4.1 Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

**Before pulling any images**, relocate Docker's data-root to the SSD:

```bash
sudo systemctl stop docker
sudo mkdir -p /mnt/ssd/docker-data
```

Create/edit `/etc/docker/daemon.json`:

```json
{
  "data-root": "/mnt/ssd/docker-data"
}
```

```bash
sudo rsync -aP /var/lib/docker/ /mnt/ssd/docker-data/ 2>/dev/null || true
sudo systemctl start docker
docker info | grep "Docker Root Dir"
```

Install the Compose plugin (bundled with `get.docker.com` on recent installs — confirm):

```bash
docker compose version
```

### 4.2 Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname=<PI-HOSTNAME>
```

> [!warning] Tailscale SSH ACL gotcha
> `tailscale set --ssh` only actually grants SSH access if the tailnet's ACL policy has a matching `ssh` accept rule. Without one, connections silently fall through to the host's regular sshd instead of erroring — it *looks* like it worked but isn't using Tailscale SSH at all. Check the tailnet admin console's ACL editor for an `"ssh"` block; add one if missing before relying on this as the emergency access path.

```bash
sudo tailscale set --ssh
```

Enable MagicDNS in the Tailscale admin console (DNS tab) if not already on — this gives every tailnet device a stable `<hostname>.<tailnet-name>.ts.net` name, independent of Pi-hole.

### Verify

```bash
docker run --rm hello-world           # Docker works, pulls from the relocated data-root
docker info | grep "Docker Root Dir"  # confirms /mnt/ssd/docker-data
tailscale status                      # Pi shows up online in the tailnet
tailscale ssh <PI-HOSTNAME>           # from another tailnet device, confirms Tailscale SSH actually works (not falling through to sshd)
```

---

## 5. Phase 2 — Pi-hole

> [!warning] This phase can take the house's DNS down
> If Pi-hole is misconfigured and the router is pointed at it, every device on the network loses name resolution. Go slowly, verify at each step, and keep the rollback plan below at hand before touching router DHCP settings.

### 5.1 Deploy

`~/stack/pihole/docker-compose.yml`:

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    hostname: <PI-HOSTNAME>
    environment:
      TZ: "America/New_York"
      FTLCONF_webserver_api_password: "<PIHOLE-WEBPASS>"
      FTLCONF_dns_upstreams: "1.1.1.1;1.0.0.1"
    volumes:
      - /mnt/ssd/<STACK-DIR>/pihole/etc-pihole:/etc/pihole
      - /mnt/ssd/<STACK-DIR>/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8081:80/tcp"
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
    networks:
      - stacknet

networks:
  stacknet:
    external: true
```

Create the shared network once, before any compose project:

```bash
docker network create stacknet
```

```bash
cd ~/stack/pihole && docker compose up -d
```

> [!tip] If port 53 is already in use
> `docker compose up` failing with "address already in use" on 53 means something on the host is already serving DNS (e.g. `systemd-resolved`'s stub listener, or dnsmasq). Find it with `sudo ss -ulpn 'sport = 53'` and disable that listener (for systemd-resolved: set `DNSStubListener=no` in `/etc/systemd/resolved.conf`, restart it, and make sure `/etc/resolv.conf` points at a real resolver) before retrying.

### 5.2 Local DNS records

Pi-hole v6 uses an API-key-authenticated REST API, separate from the web admin password shown above (they are different credentials — do not conflate them). Pattern:

```bash
# Authenticate, get a session id (sid)
SID=$(curl -s -X POST http://<PI-STATIC-IP>:8081/api/auth \
  -H "Content-Type: application/json" \
  -d '{"password":"<PIHOLE-WEBPASS>"}' | jq -r '.session.sid')

# Register a local DNS record — URL-encode "IP name" as the path segment
curl -s -X PUT "http://<PI-STATIC-IP>:8081/api/config/dns/hosts/<PI-STATIC-IP>%20<PI-HOSTNAME>" \
  -H "sid: $SID"

curl -s -X PUT "http://<PI-STATIC-IP>:8081/api/config/dns/hosts/<PI-STATIC-IP>%20<PI-HOSTNAME>.lan" \
  -H "sid: $SID"
```

Repeat for every service you want resolvable by name (register both the bare hostname and the `.lan`-suffixed form): grafana, kuma, mesh, ntfy, homepage, homarr, etc. — all pointing at `<PI-STATIC-IP>` since everything runs on the same box behind different ports, unless you prefer per-service subdomain routing via a reverse proxy (out of scope here — keep it simple, port-based).

### 5.3 Router handoff

Only after Pi-hole is confirmed working standalone (see Verify below):

- In the router admin UI, set the DNS server handed out via DHCP to `<PI-STATIC-IP>`.
- **Clients only pick up the new DNS server on their next DHCP lease renewal** — not instantly. Devices may need a reconnect or manual `ipconfig /renew` / `dhclient` to see the change immediately.

> [!tip] Rollback plan
> If Pi-hole misbehaves after the router handoff (name resolution breaks, ad-blocking is too aggressive, etc.), revert the router's DHCP DNS setting back to the ISP/router default (usually the router's own IP, which forwards to the ISP). Existing client leases will pick up the reverted DNS on their next renewal, same as the forward change. Keep this rollback step written down somewhere your human can find without you.

> [!tip] Keep MagicDNS as a fallback
> Tailscale's MagicDNS resolves tailnet hostnames independently of Pi-hole. Leave it enabled so a Pi-hole outage doesn't blind access to critical hosts reachable over the tailnet.

### Verify

```bash
dig @<PI-STATIC-IP> <PI-HOSTNAME>          # resolves to <PI-STATIC-IP>
dig @<PI-STATIC-IP> doubleclick.net        # ad domain blocked (0.0.0.0 or NXDOMAIN)
dig @<PI-STATIC-IP> google.com             # legitimate domain resolves normally
```

Do not point the router at Pi-hole until all three of these pass from a device on the LAN, not just from the Pi itself.

---

## 6. Phase 3 — Metrics core (InfluxDB + Telegraf + Grafana)

### 6.1 Deploy

`~/stack/metrics/docker-compose.yml`:

```yaml
services:
  influxdb:
    image: influxdb:2
    container_name: influxdb
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: "<INFLUX-ADMIN-PASS>"
      DOCKER_INFLUXDB_INIT_ORG: home
      DOCKER_INFLUXDB_INIT_BUCKET: telegraf
      DOCKER_INFLUXDB_INIT_RETENTION: 90d
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: "<INFLUX-ADMIN-TOKEN>"
    volumes:
      - /mnt/ssd/<STACK-DIR>/influxdb/data:/var/lib/influxdb2
      - /mnt/ssd/<STACK-DIR>/influxdb/config:/etc/influxdb2
    ports:
      - "8086:8086"
    restart: unless-stopped
    networks: [stacknet]

  telegraf:
    image: telegraf:latest
    container_name: telegraf
    user: telegraf:999   # 999 = docker group gid on most Debian-based images; confirm with `getent group docker`
    volumes:
      - /mnt/ssd/<STACK-DIR>/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/host/sys:ro
      - /proc:/host/proc:ro
    environment:
      HOST_PROC: /host/proc
      HOST_SYS: /host/sys
    depends_on: [influxdb]
    restart: unless-stopped
    networks: [stacknet]

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: "<GRAFANA-ADMIN-PASS>"
      GF_SECURITY_ALLOW_EMBEDDING: "true"
    volumes:
      - /mnt/ssd/<STACK-DIR>/grafana/data:/var/lib/grafana
      - /mnt/ssd/<STACK-DIR>/grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    depends_on: [influxdb]
    restart: unless-stopped
    networks: [stacknet]

networks:
  stacknet:
    external: true
```

`~/stack/telegraf/telegraf.conf` (portable input set — keep this identical across every host running Telegraf so measurement names line up):

```toml
[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "<INFLUX-WRITE-TOKEN>"
  organization = "home"
  bucket = "telegraf"

[[inputs.cpu]]
[[inputs.mem]]
[[inputs.disk]]
[[inputs.diskio]]
[[inputs.net]]
[[inputs.system]]
[[inputs.swap]]

[[inputs.docker]]
  endpoint = "unix:///var/run/docker.sock"

[[inputs.temp]]
  # host temperature — on most Linux boxes this surfaces via thermal zones/hwmon; confirm the metric name after first run
```

> [!warning] Two-token pattern
> Generate a **write-only** InfluxDB token scoped to the `telegraf` bucket for every Telegraf agent (`<INFLUX-WRITE-TOKEN>`), separate from the **admin** token (`<INFLUX-ADMIN-TOKEN>`) used only for Grafana's datasource setup and administrative API calls. Generate tokens from the InfluxDB UI (`http://<PI-STATIC-IP>:8086` → Data → API Tokens) or `influx auth create`.

> [!warning] Env-var token expansion only works inside Docker
> `token = "${INFLUX_TOKEN}"` env-placeholder syntax in `telegraf.conf` only expands when Telegraf is running as a Docker container with that env var passed in. A **native** Telegraf agent installed directly on another machine (see 6.3) will NOT expand this — it needs the literal token value written into its `telegraf.conf`, or it fails with 401 Unauthorized.

### 6.2 Grafana datasource

In Grafana (`http://<PI-STATIC-IP>:3000`, login `admin` / `<GRAFANA-ADMIN-PASS>`): Connections → Data sources → Add → **InfluxDB**, Query Language = **Flux**. URL `http://influxdb:8086` (container name — Grafana reaches it over `stacknet`). Organization `home`, token `<INFLUX-ADMIN-TOKEN>`, default bucket `telegraf`. Save & test. Note the datasource's UID (shown in the URL after saving) — pin it in any dashboard JSON you provision later so dashboards don't silently detach from the datasource on reprovisioning.

### 6.3 Native agents on other machines (optional, later)

Once the hub is running, other computers on the network can run a native (non-Docker) Telegraf install pushing metrics to the Pi instead of running their own InfluxDB. Use the **same portable input set** from `telegraf.conf` above (cpu/mem/disk/diskio/net/system/swap) so measurement names match across the fleet, and point `outputs.influxdb_v2.urls` at `http://<PI-STATIC-IP>:8086` with the literal (non-env-var) write token, per the warning above.

### Verify

```bash
curl -s http://<PI-STATIC-IP>:8086/health | jq   # InfluxDB reports "pass"
docker logs telegraf --tail 20                    # no auth errors, writes succeeding
```

In Grafana, Explore → InfluxDB datasource → run `from(bucket:"telegraf") |> range(start:-5m)` and confirm CPU/mem data points are flowing.

---

## 7. Phase 4 — Alerting (ntfy + Uptime Kuma)

### 7.1 ntfy

`~/stack/ntfy/docker-compose.yml`:

```yaml
services:
  ntfy:
    image: binwiederhier/ntfy
    container_name: ntfy
    command: serve
    environment:
      TZ: "America/New_York"
    volumes:
      - /mnt/ssd/<STACK-DIR>/ntfy/data:/var/cache/ntfy
      - /mnt/ssd/<STACK-DIR>/ntfy/config:/etc/ntfy
    ports:
      - "8090:80"
    restart: unless-stopped
    networks: [stacknet]
```

`~/stack/ntfy/config/server.yml`:

```yaml
base-url: "http://<PI-STATIC-IP>:8090"
auth-default-access: "deny-all"
behind-proxy: false
```

```bash
cd ~/stack/ntfy && docker compose up -d
docker exec -it ntfy ntfy user add --role=admin admin
docker exec -it ntfy ntfy user add publisher
docker exec -it ntfy ntfy access publisher "<TOPIC>" write-only
docker exec -it ntfy ntfy token add publisher
# note the returned tok_... value as <NTFY-PUBLISHER-TOKEN> — it can publish to <TOPIC> and nothing else
```

Install the ntfy app on your human's phone, subscribe to `<TOPIC>` over `http://<PI-STATIC-IP>:8090` (LAN) or the Tailscale address (remote).

### 7.2 Grafana → ntfy contact point

Grafana → Alerting → Contact points → New → type **Webhook**:

- URL: `http://ntfy/<TOPIC>` (container name — this is fetched server-side by Grafana over `stacknet`, not by the browser)
- HTTP headers: `Authorization: Bearer <NTFY-PUBLISHER-TOKEN>`

### 7.3 Uptime Kuma

`~/stack/kuma/docker-compose.yml`:

```yaml
services:
  kuma:
    image: louislam/uptime-kuma:1
    container_name: kuma
    volumes:
      - /mnt/ssd/<STACK-DIR>/kuma/data:/app/data
    ports:
      - "3001:3001"
    dns:
      - <PI-STATIC-IP>   # only needed if Kuma must resolve internal/.lan names Docker's default resolver can't
    restart: unless-stopped
    networks: [stacknet]
```

```bash
cd ~/stack/kuma && docker compose up -d
```

In the Kuma web UI (`http://<PI-STATIC-IP>:3001`), add a monitor per LAN device/service.

> [!tip] Check type by device role
> For desktops/laptops, prefer a **TCP port check by DNS name** (e.g. port 445 on Windows machines) over ICMP ping — host firewalls commonly block ping, giving false "down" alerts. Attach **no notifications** to routine user machines (laptops, phones-that-sleep) to avoid alert spam; reserve notifications for services that should always be up (Pi-hole, Grafana, the router, servers).

Add ntfy as a Notification (Settings → Notifications → new ntfy-type notification, server URL + topic + access token), then attach it to the monitors that should page.

### 7.4 UPS monitoring (NUT) — if a UPS is present

If this stack runs in a VM, run NUT on the hypervisor host instead and skip this subsection in the guest.

If the Pi (or the desk it protects) has a USB-connected UPS (e.g. a CyberPower CP1500PFCLCD-class unit), run Network UPS Tools **natively on the Pi**, not in Docker — USB device passthrough adds complexity for no benefit here.

```bash
sudo apt install -y nut
```

- Driver: `usbhid-ups` (covers CyberPower/APC USB models).
- `/etc/nut/ups.conf`:

```ini
[homeups]
	driver = usbhid-ups
	port = auto
```

- Run `upsd` + `upsmon` in standalone mode (the default single-machine setup — see `/etc/nut/nut.conf`, `MODE=standalone`).

Add Telegraf coverage — append to the Pi's `telegraf.conf`:

```toml
[[inputs.upsd]]
  server = "127.0.0.1:3493"
```

This lands battery charge, load, runtime, and input voltage in InfluxDB. Suggest a Grafana panel row for battery %, estimated runtime, and load (W).

Wire ntfy on power events with an `upsmon` NOTIFYCMD wrapper script (or `upssched`) that fires on `ONBATT` / `LOWBATT` / `ONLINE` transitions:

```bash
#!/bin/bash
# /etc/nut/notify.sh
curl -H "Authorization: Bearer <NTFY-PUBLISHER-TOKEN>" -d "UPS: $1" http://localhost:8090/<TOPIC>
```

Point `NOTIFYCMD` in `/etc/nut/upsmon.conf` at this script (`chmod +x` it first), and add a `NOTIFYFLAG <event> SYSLOG+EXEC` line for each event that should fire it (`ONBATT`, `LOWBATT`, `ONLINE`) — without the `EXEC` flag, `NOTIFYCMD` is never invoked.

> [!tip] Optional fleet graceful shutdown
> Other always-on machines can run the NUT netclient pointed at the Pi's `upsd` (`MONITOR homeups@<PI-STATIC-IP> 1 monuser <pass> slave` in their `upsmon.conf`), so a real outage triggers ordered shutdowns — clients go down first, the Pi last. Prerequisites on the Pi first: define `monuser` (with `upsmon slave`) in `/etc/nut/upsd.users`, and add `LISTEN <PI-STATIC-IP>` to `/etc/nut/upsd.conf` — `upsd` listens on localhost only by default, so remote clients can't connect until then.

### Verify

```bash
upsc homeups@localhost | grep battery.charge   # returns a number
```

Then run a supervised drill: with your human present, unplug the UPS from wall power for ~30 seconds and confirm a phone push arrives, then plug it back in and confirm a recovery push.

> [!warning] Never run the unplug drill unattended
> Do this only with your human present. Pulling a UPS from wall power is a real power-loss test for whatever it protects.

---

## 8. Phase 5 — MeshCentral

`~/stack/meshcentral/docker-compose.yml`:

```yaml
services:
  meshcentral:
    image: typhonragewind/meshcentral:latest
    container_name: meshcentral
    environment:
      HOSTNAME: "<MESHCENTRAL-CERT-HOST>"
    volumes:
      - /mnt/ssd/<STACK-DIR>/meshcentral/data:/opt/meshcentral/meshcentral-data
    ports:
      - "443:443"
      - "4433:4433"
    restart: unless-stopped
    networks: [stacknet]
```

> [!warning] meshcentral-data is not optional
> The `meshcentral-data` volume holds the server's TLS certs and identity. Losing it orphans every agent that's already been installed — they will never reconnect to a freshly generated cert. Back this directory up (see Phase 7).

`config.json` (inside `meshcentral-data`, generated on first run — edit after):

```json
{
  "settings": {
    "cert": "<MESHCENTRAL-CERT-HOST>",
    "WANonly": true
  },
  "domains": {}
}
```

> [!warning] Never use LAN mode
> LAN mode bakes local multicast discovery (`MeshServer=local`) into installers and agent configs. Agents built this way will never connect to the server across subnets or over the tailnet — they only find a server on the same broadcast domain. Always use `WANonly: true`.

> [!warning] Never hand-build a `.msh` agent config
> The `ServerID` field in an agent's `.msh` config is the **SHA-384 hash of the agent certificate's public key** — not a hash of the whole cert, not a copy-pasted string from somewhere else. Getting this wrong produces a genuinely nasty silent failure: the server log says "Verified agent connection," the agent itself rejects the server and loops forever, zero devices show up in the console, and there is no clear error pointing at the cause. **Always generate agent installers from the server** — download them from the MeshCentral web UI's "My Devices" → device group → Agent download, or via `meshctrl AgentDownload` — so `ServerID` and the connection URL are baked in correctly automatically.

### Setup

1. Set `<MESHCENTRAL-CERT-HOST>` to a stable address before agents are ever installed — changing it later breaks every existing agent. The Pi's Tailscale MagicDNS name (e.g. `<PI-HOSTNAME>.<TAILNET-NAME>.ts.net`) is ideal since it works whether an agent is home or away. A LAN IP/hostname is acceptable only if every managed device will always be on the home network.
2. In the web UI, create one device group (e.g. "mesh").
3. Download the server-baked agent installer for that group and install it on each machine identified in Section 2, Question 5.

### Verify

```bash
docker logs meshcentral --tail 30   # no cert errors, server listening on 443
```

In the web UI, confirm each installed agent shows as connected/online in the device group within a few minutes of installation. If an agent never appears, do not hand-edit its `.msh` — regenerate the installer from the server and reinstall.

---

## 9. Phase 6 — Dashboards (Homepage + Homarr)

> [!warning] The dual-URL rule — read this before wiring any widget
> A widget's `url:` field is fetched **server-side**, from inside the dashboard's own container — it must use the **docker-network container name** (e.g. `http://pihole`, `http://grafana:3000`), which only resolves because the dashboard containers share `stacknet` with the services. A widget's `href:` (or an iframe's `src:`) is loaded **by the browser** on your human's device — it must use the **LAN hostname or IP** (e.g. `http://<PI-STATIC-IP>:8081`), because the browser has no access to the docker-internal network namespace. Mixing these two up is the single most common cause of "dashboard looks broken" — a widget with a container-name `href` gives a dead link in the browser; a widget with a LAN-IP `url` (server-side fetch) may still work by accident if the LAN IP happens to be reachable from inside the container, or may fail depending on host networking — don't rely on that, use the right one for the right field.

### 9.1 Homepage

`~/stack/homepage/docker-compose.yml`:

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:v1.13.2
    container_name: homepage
    environment:
      HOMEPAGE_ALLOWED_HOSTS: "<PI-STATIC-IP>:8082,<PI-HOSTNAME>:8082,<PI-HOSTNAME>.lan:8082"
    volumes:
      - /mnt/ssd/<STACK-DIR>/homepage/config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "8082:3000"
    restart: unless-stopped
    networks: [stacknet]
```

> [!warning] HOMEPAGE_ALLOWED_HOSTS
> Homepage 400s with "Host validation failed" for any host:port combination not explicitly listed here. List **every** address it will be browsed from — LAN IP, hostname, `.lan` suffix, and the Tailscale MagicDNS name if it'll be reached remotely.

`~/stack/homepage/config/services.yaml` (example — server-side `url` uses container names):

```yaml
- Network:
    - Pi-hole:
        icon: pi-hole.png
        href: http://<PI-STATIC-IP>:8081/admin
        widget:
          type: pihole
          url: http://pihole
          key: <PIHOLE-API-KEY>   # generated in Pi-hole's web UI, Settings → API — different from PIHOLE-WEBPASS
    - Grafana:
        icon: grafana.png
        href: http://<PI-STATIC-IP>:3000
        widget:
          type: grafana
          url: http://grafana:3000
          username: admin
          password: <GRAFANA-ADMIN-PASS>
    - Uptime Kuma:
        icon: uptime-kuma.png
        href: http://<PI-STATIC-IP>:3001
        widget:
          type: uptimekuma
          url: http://kuma:3001
          slug: <KUMA-STATUS-PAGE-SLUG>
    - MeshCentral:
        icon: si-meshcentral
        href: https://<MESHCENTRAL-CERT-HOST>
    - ntfy:
        icon: ntfy.png
        href: http://<PI-STATIC-IP>:8090
```

`~/stack/homepage/config/settings.yaml` — set title/theme as your human prefers.

### 9.2 Homarr

`~/stack/homarr/docker-compose.yml`:

```yaml
services:
  homarr:
    image: ghcr.io/homarr-labs/homarr:latest
    container_name: homarr
    volumes:
      - /mnt/ssd/<STACK-DIR>/homarr/appdata:/appdata
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "7575:7575"
    restart: unless-stopped
    networks: [stacknet]
```

> [!tip] Homarr appdata is root-owned by default
> If your human needs host-side access to the sqlite state under `appdata`, fix ownership with a throwaway container rather than `sudo chown` on the host if permissions get confusing: `docker run --rm -v /mnt/ssd/<STACK-DIR>/homarr/appdata:/data alpine chown -R 1000:1000 /data`.

Configure Homarr entirely through its web UI (`http://<PI-STATIC-IP>:7575`) — add the same five services (Pi-hole, Grafana, Kuma, MeshCentral, ntfy) as apps/widgets, respecting the same dual-URL rule as Homepage.

### 9.3 Grafana embedding

For either dashboard to embed a live Grafana panel/dashboard via iframe, the URL form is:

```
http://<PI-STATIC-IP>:3000/d/<DASHBOARD-UID>/<SLUG>?orgId=1&kiosk&theme=dark&refresh=30s&from=now-6h&to=now
```

This requires `GF_SECURITY_ALLOW_EMBEDDING=true` (already set in the Phase 3 compose file). If your human wants the wallboard to load without a Grafana login prompt, also enable anonymous Viewer access in Grafana's auth settings — a deliberate tradeoff between convenience and access control, flag it before enabling.

> [!warning] Mixed content
> If the dashboard page is ever served over HTTPS, an HTTP Grafana iframe will be blocked by the browser as mixed content. Keep everything on plain HTTP for a LAN-only wallboard, or put both behind HTTPS consistently — don't mix.

### 9.4 Kiosk wallboard (optional)

If your human has a secondary/glanceable display (a wall-mounted panel, or a spare monitor half), build one Grafana dashboard named "Wallboard" aggregating: fleet CPU/mem, GPU utilization (if Phase 8's GPU agents exist), UPS battery (if 7.4 was adopted), and disk headroom. Embed it, or bookmark it directly, using the kiosk URL form already documented in 9.3. Uptime Kuma's own status page (`http://<PI-STATIC-IP>:3001/status/<KUMA-STATUS-PAGE-SLUG>`) makes a good companion tab.

Whatever machine drives that display just needs a browser full-screening the kiosk URL — the same anonymous-Viewer access tradeoff from 9.3 applies here too.

### Verify

Open both dashboards from a LAN browser and confirm: Pi-hole widget shows live block counts, Grafana widget/iframe renders a real panel, Kuma widget shows monitor status, MeshCentral and ntfy links open correctly, and Docker container-status indicators are populated (confirms the ro socket mount works). If 9.4 was built, confirm the Wallboard dashboard renders on the kiosk display itself, not just in a regular browser tab.

---

## 10. Phase 7 — Verification appendix + maintenance

### 10.1 Per-phase re-check table

| Phase | Quick health check |
|---|---|
| 0 | `df -h /mnt/ssd` mounted; `dmesg \| grep -i uas` clean |
| 1 | `docker info \| grep "Docker Root Dir"` on SSD; `tailscale status` online |
| 2 | `dig @<PI-STATIC-IP> <PI-HOSTNAME>` resolves; ad domain blocked |
| 3 | `curl http://<PI-STATIC-IP>:8086/health` passes; Grafana Explore shows data |
| 4 | Stop a container → Kuma red → phone push arrives; if 7.4 adopted, `upsc homeups@localhost` returns a battery.charge number |
| 5 | Agents show connected in MeshCentral device group |
| 6 | Both dashboards render live widget data, no dead links |
| 8 | Fleet machines' hostname tags visible in InfluxDB Data Explorer within 2 minutes |

### 10.2 End-to-end drill

Run this after all phases are up, as the final sign-off:

```bash
docker stop pihole
```

Confirm, in order: Uptime Kuma marks Pi-hole down within its check interval → ntfy push arrives on the phone → Homepage/Homarr container-status widget shows it stopped. Then:

```bash
docker start pihole
```

Confirm the reverse: Kuma goes green, a "back up" notification arrives if configured, dashboards update.

### 10.3 Restart-policy gotcha

> [!warning] Editing `restart:` in a compose file does not affect a running container
> Changing `restart: unless-stopped` in a `docker-compose.yml` for an already-running container has no effect until the container is recreated. To apply a restart-policy change to a live container without recreating it:
> ```bash
> docker update --restart=unless-stopped <container-name>
> ```
> Verify with:
> ```bash
> docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' <container-name>
> ```

### 10.4 Updates

For each compose project directory:

```bash
docker compose pull
docker compose up -d
```

### 10.5 Backups

Back up the full stack directory regularly, and get a copy off the Pi itself (a second drive, another machine on the network, or cloud storage — your human's call):

```bash
tar -czf ~/stack-backup-$(date +%F).tar.gz -C /mnt/ssd <STACK-DIR>
```

This tarball includes `meshcentral/data` (agent certs — irreplaceable), `kuma/data` (monitor history/config), `grafana/provisioning` + `grafana/data` (dashboards, datasources), and `/etc/nut` on the Pi itself (UPS config — see 7.4, if adopted, back this up separately since it's outside `/mnt/ssd`). Treat `meshcentral/data` as the single most important thing in this backup — losing it orphans every managed agent.

---

## 11. Phase 8 — Fleet agents (other machines, per OS)

Once the hub (Phases 0–7) is running, extend native Telegraf coverage to other machines on the network — all pushing to the Pi at `http://<PI-STATIC-IP>:8086` with the literal (non-env-var) write token; the env-var expansion gotcha from 6.1 applies here too, since none of these are Docker containers.

> [!warning] Work machines are excluded
> Per Section 2, Question 8: never install any agent from this section — Telegraf, MeshCentral, or otherwise — on a work-issued or employer-owned machine. The most that's permissible there is a passive Kuma reachability check.

### 11.1 macOS

```bash
brew install telegraf
```

Config lives at `/opt/homebrew/etc/telegraf.conf` (Apple Silicon). Use the same portable input set from 6.1 (cpu/mem/disk/diskio/net/system/swap) so measurement names stay consistent across the fleet.

```bash
brew services start telegraf
```

### 11.2 Windows

Install via the official zip/MSI as a service. The portable input set works cross-platform — keep measurement-name parity with the Pi's own config. Telegraf on Windows supports a config-directory pattern for drop-in `.conf` files if you need to layer machine-specific inputs on top of the shared base.

> [!warning] Gaming-machine GPU polling causes stutter
> Polling GPU metrics (`nvidia_smi` input, LibreHardwareMonitor) at short intervals causes visible in-game stutter — verified on a production gaming rig. On any machine used for gaming, either poll GPU inputs at 300s or skip them entirely. The cheap gopsutil-based inputs (cpu/mem/disk/net) are safe at default intervals.

### 11.3 Linux x86/ARM64 (including NVIDIA AI boxes — DGX Spark class)

Use the influxdata apt repo.

> [!warning] Debian 13+ keyring gotcha
> Use the `influxdata-archive.key`, not the older compatibility key — the compat key is rejected on Debian 13+.

Use the same portable input set as everywhere else. On NVIDIA boxes, add:

```toml
[[inputs.nvidia_smi]]
```

This surfaces GPU utilization, VRAM, temperature, and power. A headless compute box has no stutter concern — a 60s interval is fine. Suggest a per-GPU Grafana row for these.

### 11.4 MeshCentral agents

Supported on all three OS families, including Linux arm64. Always install via the server-baked installer per Phase 5 — never hand-build a `.msh` config.

> [!tip] Wake-on-LAN for free
> MeshCentral can wake sleeping machines on the same LAN. Enable WoL in each machine's firmware/adapter settings to get remote wake for free once the agent is installed.

### 11.5 Kuma coverage

Add one Uptime Kuma monitor per fleet machine, per the check-type guidance in 7.3 (TCP-by-DNS-name preferred over ping; no notifications on routine user machines).

### Verify

From each agented machine, confirm in InfluxDB's Data Explorer that its hostname tag appears within 2 minutes of the agent starting. For NVIDIA boxes, confirm GPU measurements are present alongside the standard set.

---

*End of charter. If anything here conflicts with what you find on the actual hardware/network, stop and ask your human rather than guessing past the discrepancy.*
