# Hawking Home Lab

A rebuild guide for a Dell Wyse 5070 running Debian, Home Assistant, and self-hosted network and media services. The repository is organized around a GitHub Pages site in `docs/` and reusable, sanitized configuration examples.

## Service inventory

| Component | Current setup |
| --- | --- |
| Host | Debian with ifupdown and DHCP. The SSD holds the OS and service data. The HDD mounts at `/mnt/storage`. |
| Home Assistant | Docker image `ghcr.io/home-assistant/home-assistant:stable`, host networking, privileged mode, a writable config bind mount at `/config`, and read-only `/run/dbus`. Restarts unless stopped. |
| Matter Server | Docker image `ghcr.io/home-assistant-libs/python-matter-server:stable`, host networking, and an external volume at `/data`. The PAA root certificate directory is `/data/credentials`. A systemd unit starts Compose. Docker restart policy is `no`. |
| Wyoming Piper | Docker image `rhasspy/wyoming-piper:latest`, bridge networking, TCP `10200`, named volume `piper-data` at `/data`, and voice `en_US-lessac-medium`. Restarts unless stopped. |
| Wyoming Whisper | Docker image `rhasspy/wyoming-whisper:latest`, bridge networking and TCP `10300`. The running container has no explicit startup arguments or mounts. Restarts unless stopped. |
| Native services | Nginx, Pi-hole, Jellyfin, and Tailscale. Host monitoring uses `go-hass-agent` as a systemd user service. |
| ESPHome | Builds run on the desktop. The old server container is stopped. Desktop setup and device YAML will be added later, starting with the SCD41-equipped finch sensor. |
| Clients | Home Assistant clients include a Garmin app that uses the Tailscale HTTPS hostname. |

## Network access

| Entry point | Route |
| --- | --- |
| Tailnet HTTPS on `443` | Tailscale Serve forwards to Nginx on loopback port `80`. Nginx has configured locations for Jellyfin, Home Assistant, and Pi-hole. |
| Tailnet HTTPS on `8443` | Tailscale Serve forwards directly to Home Assistant on loopback port `8123`. |
| LAN HTTPS on `8443` | Nginx listens on the LAN address, presents the certificate for the Tailscale hostname, and proxies to Home Assistant. Pi-hole can resolve the same hostname to the LAN address for clients using its local DNS record. |
| Local Jellyfin and Home Assistant names | Nginx listens on the LAN address at `443` and proxies to `8096` and `8123`, respectively. The Jellyfin certificate was generated with mkcert. Client trust and the issuance method for the other local certificate still need confirmation. |
| Pi-hole web interface | Nginx redirects `/pihole` to `/admin/` and proxies `/admin/` and `/api/` to Pi-hole on `8080`. |

Jellyfin is installed natively at `10.11.11+deb13` with `jellyfin-ffmpeg7` at `7.1.4-3-trixie`. Its Base URL is **`/jellyfin`**. The systemd service uses `Restart=on-failure`.

Pi-hole has four local DNS records pointing to the server's LAN address. They cover the Tailscale hostname and local names for Home Assistant, Jellyfin, and Pi-hole. Its DNS listening mode is **Permit all origins**. The network guide should document this alongside the intended DNS clients and firewall restrictions. The firewall guide should preserve the ordered UFW rules, including IPv4 and IPv6 forwarding from the Tailscale interface to the LAN interface.

## Power-outage startup

| Component | Startup behavior |
| --- | --- |
| Debian networking | `ifupdown-wait-online.service` is enabled and runs the Debian wait script before `network-online.target`. |
| Nginx | A systemd drop-in requires `networking.service`, waits for the configured LAN address, then validates the Nginx config before starting. |
| Matter Server | An enabled systemd oneshot unit waits for Docker, checks for the LAN address, and resolves `api.github.com` before running Compose in detached mode. Startup has a 120-second timeout. |

The checks above run at startup. A delayed-network boot and a network interruption after startup are separate verification cases. The storage guide should also test what happens when the HDD is absent or slow to mount, especially for media and file sharing.

## Planned repository layout

| Path | Contents |
| --- | --- |
| `docs/index.md` | GitHub Pages homepage with an architecture diagram and rebuild checklist |
| `docs/host.md` | Hardware, Debian, DHCP, and the UUID-based `/mnt/storage` mount |
| `docs/containers.md` | Home Assistant, Matter Server, Piper, and Whisper deployment and persistence |
| `docs/native-services.md` | Pi-hole, Jellyfin, host monitoring, and Samba if the reference share matches the live setup |
| `docs/networking.md` | Local DNS, Nginx, Tailscale Serve, HTTPS certificates, and client trust |
| `docs/firewall.md` | Ordered UFW rules, LAN and tailnet access, and forwarding |
| `docs/home-assistant.md` | Matter, Wyoming voice pipeline, Garmin access, and integrations |
| `docs/devices/` | Desktop ESPHome workflow, device wiring, YAML, and pairing |
| `docs/startup-recovery.md` | Power-outage startup, systemd dependencies, and recovery tests |
| `docs/operations.md` | Backups, restore order, upgrades, and troubleshooting |
| `deploy/docker/` | Sanitized launch examples and instructions for creating or reusing the Matter volume |
| `deploy/host/` | Nginx site templates, systemd units and drop-ins, UFW rules, and an example storage mount |
| `config/home-assistant/`, `config/esphome/` | Shareable YAML and placeholder secrets templates |
| `scripts/` | Optional SMART disk activity monitor from the server setup archive, after checking its standby behavior |

## Rebuild order

1. Install Debian, configure DHCP and storage, and verify the HDD mount.
2. Install Docker and prepare the Home Assistant config directory and Matter volume. Start Home Assistant, Piper, and Whisper. Install the Matter systemd unit.
3. Set up Nginx, Pi-hole, Jellyfin, Tailscale, and UFW. Verify SSH access before and after applying the firewall rules.
4. Recreate Home Assistant integrations and the voice pipeline. Add ESPHome from the desktop when the device guides are available.
5. Verify service access from the LAN and tailnet. Test startup with delayed networking and verify the backup and restore procedure.

## Details to verify

- Jellyfin media directories and hardware transcoding settings.
- Home Assistant proxy settings and the Wyoming integration and voice pipeline configuration.
- Pi-hole upstream DNS, client DNS distribution, and any Tailscale split DNS.
- UFW default policies, Samba profile ports, router port forwards, and IoT network isolation.
- Which clients trust the mkcert CA, the exact Garmin URL in use, and which legacy Nginx paths are still used.
- Current Samba share, backup and restore procedure, and desktop ESPHome setup.

## Public examples

Use placeholders such as `<SERVER_LAN_IP>`, `<LAN_CIDR>`, `<TAILNET_HOST>`, and `<MATTER_VOLUME>`. Keep real addresses, usernames, private keys, tokens, Matter pairing codes, live environment files, raw exports, databases, and backups out of commits and screenshots. Show certificate paths and setup steps with placeholders.
