# AWG 2.0 Easy

The easiest way to run **AmneziaWG 2.0** with a Web UI on any Linux host.

Built on [amnezia-wg-easy](https://github.com/w0rng/amnezia-wg-easy), this image swaps the base to `amneziavpn/amneziawg-go` and adds full AWG 2.0 obfuscation parameters (I1–I5, S3, S4) alongside the existing AWG 1.x fields (Jc, Jmin, Jmax, S1, S2, H1–H4).

## Features

* AmneziaWG 2.0 (Go userspace) + Web UI, all in one image.
* Full AWG 2.0 config generation — I1–I5, S3, S4 included in every client config.
* List, create, edit, delete, enable & disable clients.
* Show a client's QR code.
* Download a client's configuration file.
* Statistics for which clients are connected.
* Tx/Rx charts for each connected client.
* Gravatar support or random avatars.
* Automatic Light / Dark Mode.
* Multilanguage Support.
* Traffic Stats (default off).
* One Time Links (default off).
* Client Expiry (default off).
* Prometheus metrics support.

## Requirements

* A host with Docker and Docker Compose installed.

## Installation

### 1. Install Docker

```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker $(whoami)
exit
```

Log back in after running the above.

### 2. Generate a password hash

```bash
docker run --rm node:18-alpine sh -c \
  "npm install bcryptjs --silent && \
   node -e \"require('bcryptjs').hash('YOUR_PASSWORD',12).then(h=>console.log(h))\""
```

### 3. Create a `docker-compose.yml`

```yaml
services:
  awg2-easy:
    image: ghcr.io/FromMun/awg2-easy:latest
    container_name: awg2-easy
    environment:
      - WG_HOST=YOUR_SERVER_IP
      - WG_PORT=51820
      - WG_CONFIG_PORT=51820
      - WG_PATH=/etc/amnezia/amneziawg
      - PORT=51821
      - WG_DEFAULT_ADDRESS=10.8.0.x
      - PASSWORD_HASH=YOUR_BCRYPT_HASH
    volumes:
      - ./data:/etc/amnezia/amneziawg
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    devices:
      - /dev/net/tun:/dev/net/tun
```

> Replace `YOUR_SERVER_IP` with your public IP or hostname.  
> Replace `YOUR_BCRYPT_HASH` with the hash generated in step 2.  
> **Note:** Dollar signs in `PASSWORD_HASH` must be escaped as `$$` when set inline in `docker-compose.yml`. To avoid this, put the value in a `.env` file instead.
>
> **Port quick-reference:**  
> `PORT` (`51821/tcp`) is the **Web UI** — open it in your browser.  
> `WG_PORT` (`51820/udp`) is the **VPN tunnel** — peers connect here.  
> `WG_CONFIG_PORT` only needs to be set if a firewall/NAT translates the external UDP port before it reaches the container (e.g. external `51825` → internal `51820` → set `WG_CONFIG_PORT=51825`).

### 4. Start

```bash
docker compose up -d
```

The Web UI will be available at `http://YOUR_SERVER_IP:51821`.

## Running multiple instances

Each instance needs a unique `WG_INTERFACE`, `WG_PORT`, `WG_CONFIG_PORT`, and `PORT`. Everything else — config files, iptables rules, and the Web UI session cookie — is derived automatically from `WG_INTERFACE`.

```yaml
services:
  awg2-easy-a:
    image: ghcr.io/FromMun/awg2-easy:latest
    environment:
      - WG_HOST=YOUR_SERVER_IP
      - WG_INTERFACE=awg0
      - WG_PORT=51820
      - WG_CONFIG_PORT=51820
      - PORT=51821
      - WG_PATH=/etc/amnezia/amneziawg0
      - PASSWORD_HASH=YOUR_BCRYPT_HASH
    volumes:
      - ./data0:/etc/amnezia/amneziawg0
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    # ... cap_add, sysctls, devices

  awg2-easy-b:
    image: ghcr.io/FromMun/awg2-easy:latest
    environment:
      - WG_HOST=YOUR_SERVER_IP
      - WG_INTERFACE=awg1
      - WG_PORT=51830
      - WG_CONFIG_PORT=51830
      - PORT=51831
      - WG_DEFAULT_ADDRESS=10.9.0.x
      - WG_PATH=/etc/amnezia/amneziawg1
      - PASSWORD_HASH=YOUR_BCRYPT_HASH
    volumes:
      - ./data1:/etc/amnezia/amneziawg1
    ports:
      - "51830:51830/udp"
      - "51831:51831/tcp"
    # ... cap_add, sysctls, devices
```

> Each instance must also use a different `WG_DEFAULT_ADDRESS` subnet (e.g. `10.8.0.x` and `10.9.0.x`) to avoid routing conflicts.

## Options

| Env | Default | Description |
|-----|---------|-------------|
| `WG_HOST` | — | Public IP or hostname of your VPN server. **Required.** |
| `WG_INTERFACE` | `awg0` | Kernel network interface name. Change this when running multiple instances on the same host to avoid conflicts (e.g. `awg1`). Also determines the session cookie name. |
| `PORT` | `51821` | TCP port for the Web UI. |
| `WEBUI_HOST` | `0.0.0.0` | IP address the Web UI binds to. |
| `PASSWORD_HASH` | — | Bcrypt hash for Web UI login. If unset, no password is required. |
| `WG_DEVICE` | `eth0` | Network interface traffic is masqueraded through. |
| `WG_PORT` | `51820` | **VPN UDP port** — AmneziaWG listens on this port inside the container. Must match the right-hand side of your `ports` mapping (e.g. `"51820:51820/udp"`). |
| `WG_CONFIG_PORT` | `WG_PORT` | **Client endpoint port** — written into downloaded client `.conf` files as the `Endpoint` port. Only set this when a firewall or NAT translates the external port before it reaches the container. Example: external port `51825` mapped to internal `51820` → set `WG_CONFIG_PORT=51825`. If unset, equals `WG_PORT`. |
| `WG_MTU` | — | MTU for clients. |
| `WG_PERSISTENT_KEEPALIVE` | `0` | Keepalive interval in seconds (`0` = disabled). |
| `WG_DEFAULT_ADDRESS` | `10.8.0.x` | Client IP range. |
| `WG_DEFAULT_DNS` | `1.1.1.1` | DNS server pushed to clients. |
| `WG_ALLOWED_IPS` | `0.0.0.0/0, ::/0` | Allowed IPs pushed to clients. |
| `WG_PRE_UP` | — | Command run before the interface comes up. |
| `WG_POST_UP` | — | Command run after the interface comes up. |
| `WG_PRE_DOWN` | — | Command run before the interface goes down. |
| `WG_POST_DOWN` | — | Command run after the interface goes down. |
| `LANG` | `en` | Web UI language (en, ru, de, fr, es, tr, pl, and more). |
| `UI_TRAFFIC_STATS` | `false` | Show per-client Tx/Rx stats. |
| `UI_CHART_TYPE` | `0` | `0` = off, `1` = line, `2` = area, `3` = bar. |
| `WG_ENABLE_ONE_TIME_LINKS` | `false` | Enable one-time download links (expire after 5 min). |
| `WG_ENABLE_EXPIRES_TIME` | `false` | Enable client expiry dates. |
| `MAX_AGE` | `0` | Web UI session lifetime in minutes (`0` = until browser closes). |
| `UI_ENABLE_SORT_CLIENTS` | `false` | Sort clients by name in the UI. |
| `ENABLE_PROMETHEUS_METRICS` | `false` | Expose `/metrics` and `/metrics/json`. |
| `PROMETHEUS_METRICS_PASSWORD` | — | Bcrypt hash for Prometheus Basic Auth. |
| `DICEBEAR_TYPE` | `false` | Avatar style (see [dicebear.com](https://www.dicebear.com/styles/)). |
| `USE_GRAVATAR` | `false` | Use Gravatar avatars. |

### AWG 1.x obfuscation parameters

| Env | Default | Description |
|-----|---------|-------------|
| `JC` | random (3–10) | Junk packet count. |
| `JMIN` | `50` | Junk packet minimum size (bytes). |
| `JMAX` | `1000` | Junk packet maximum size (bytes). |
| `S1` | random | Init packet junk size (bytes). |
| `S2` | random | Response packet junk size (bytes). |
| `H1`–`H4` | random | Magic header values (must be < uint32 max). |

### AWG 2.0 obfuscation parameters

| Env | Default | Description |
|-----|---------|-------------|
| `S3` | random | Additional init packet junk size (bytes). |
| `S4` | random | Additional response packet junk size (bytes). |
| `I1`–`I5` | random | Initialization packet sizes (bytes, 1–1500). |

## Updating

```bash
docker compose pull
docker compose up -d
```

## Thanks

Based on [amnezia-wg-easy](https://github.com/w0rng/amnezia-wg-easy) by w0rng.  
Originally derived from [wg-easy](https://github.com/wg-easy/wg-easy) by Emile Nijssen.  
AmneziaWG integration from [amnezia-wg-easy](https://github.com/spcfox/amnezia-wg-easy) by Viktor Yudov.
