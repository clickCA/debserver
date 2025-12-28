# VPN Port Forwarding with trueDDNS (Limited Port Range Setup)

## Overview

This guide walks through setting up a WireGuard VPN using **trueDDNS** when you only have access to a **small port range (11990–11999)**.
This situation is common with ISPs like **True Internet**, especially if you’re behind **CGNAT** or don’t have a full public IP.

The goal here is simple:
run a reliable VPN at home using Docker, forward only the ports you’re allowed to use, and still be able to connect from anywhere.

---

## What You’ll Need

Before starting, make sure you have:

- A True Internet connection with **trueDDNS** enabled
- A router that supports **port forwarding**
- A server or machine running **Docker + Docker Compose**
- Access to ports **11990–11999** (UDP/TCP as required)

---

## Understanding the Limited Port Range

TrueDDNS usually gives you a **small fixed range of ports**, in this case:

- **11990–11999** → 10 usable ports total

How we’ll use them:

- **11990 (UDP)** → WireGuard VPN traffic
- **51821 (TCP)** → wg-easy Web UI
- **11991–11999** → Optional future services or additional VPN setups

Each VPN client connects through the same UDP port (11990), so you don’t need one port per device.

---

## Step 1: Set Up trueDDNS

First, register a hostname with trueDDNS.

Example:

```
yourdomain.trueddns.com
```

You can do this via the True Internet / trueDDNS portal:

```
https://trueddns.com/
```

This hostname will always point to your home network, even if your IP changes.

---

## Step 2: Run WireGuard Using Docker (wg-easy)

We’ll use **wg-easy**, which gives you:

- A clean Web UI
- Easy client management
- Automatic WireGuard config generation

Create a `docker-compose.yml` (or `compose.yaml`) file:

```yaml
volumes:
  etc_wireguard:

services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy

    environment:
      - LANG=en
      - WG_HOST=yourdomain.trueddns.com

      # Web UI port (TCP)
      - PORT=51821

      # WireGuard UDP port (must match forwarded port)
      - WG_PORT=11990

      - WG_DEFAULT_DNS=4.4.4.4,8.8.8.8,1.1.1.1
      - WG_ALLOWED_IPS=0.0.0.0/0,192.168.1.0/24,10.0.1.0/24

    volumes:
      - etc_wireguard:/etc/wireguard

    ports:
      - "11990:11990/udp"
      - "51821:51821/tcp"

    restart: unless-stopped

    cap_add:
      - NET_ADMIN
      - SYS_MODULE

    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```

Start the container:

```bash
docker compose up -d
```

After this, WireGuard is running — but not reachable yet until we forward ports.

---

## Step 3: Configure Router Port Forwarding

Log in to your router’s admin panel and add these rules.

### WireGuard (VPN traffic)

- **External Port**: 11990
- **Internal Port**: 11990
- **Internal IP**: IP of your Docker server
- **Protocol**: UDP

### wg-easy Web UI

- **External Port**: 51821
- **Internal Port**: 51821
- **Internal IP**: IP of your Docker server
- **Protocol**: TCP

### Example Port Usage

| Port        | Purpose             |
| ----------- | ------------------- |
| 11990       | WireGuard VPN (UDP) |
| 11991       | Free / future use   |
| 11992–11999 | Reserved            |

---

## Step 4: Firewall Rules (Server Side)

Make sure the server allows traffic on these ports.

### UFW

```bash
sudo ufw allow 11990/udp comment 'WireGuard VPN'
sudo ufw allow 51821/tcp comment 'WG-Easy Web UI'
```

### iptables

```bash
sudo iptables -A INPUT -p udp --dport 11990 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 51821 -j ACCEPT
```

---

## Step 5: Access wg-easy and Create VPN Clients

Open your browser:

```
http://yourdomain.trueddns.com:51821
```

(or use the server IP if testing locally)

From the UI:

1. Click **New Client**
2. Name it (e.g. `iphone`, `laptop`, `work-pc`)
3. Click **Create**
4. Scan the QR code or download the config file

That’s it — no manual WireGuard config needed.

---

## Step 6: Client Setup

### Mobile (iOS / Android)

1. Install the **WireGuard** app
2. Scan the QR code
3. Toggle **Connect**

### Desktop (Windows / macOS / Linux)

1. Import the downloaded `.conf` file
2. Connect using the WireGuard client

The endpoint will look like:

```
yourdomain.trueddns.com:11990
```

---

## Testing the Setup

### Check if the Port Is Listening

```bash
sudo ss -tuln | grep 11990
```

### Test From Outside Your Network

```bash
nc -uzv yourdomain.trueddns.com 11990
```

Check the Web UI:

```bash
curl http://yourdomain.trueddns.com:51821
```

---

## Troubleshooting

### Can’t Connect?

Start with these checks:

1. **Router forwarding** is correct (UDP vs TCP matters)
2. **Firewall** isn’t blocking the ports
3. **Container is running**:

   ```bash
   docker ps
   docker logs wg-easy
   ```

4. **CGNAT**: Some True Internet plans still block inbound traffic
5. **ISP blocking**: Confirm port availability with True support

---

### trueDDNS Not Resolving

Check DNS:

```bash
nslookup yourdomain.trueddns.com
```

If you changed the hostname, recreate the container:

```bash
docker compose down
docker compose up -d
```

---

### Connection Drops Randomly

- Ensure `WG_PERSISTENT_KEEPALIVE=25` (default)
- Verify `WG_ALLOWED_IPS`
- Check router NAT timeout settings
- Review client logs in the WireGuard app

---

## Security Tips (Don’t Skip These)

- Use strong passwords or client keys
- Don’t expose unnecessary ports
- Keep Docker images updated
- Monitor logs occasionally
- Restrict Web UI access if possible

---

## If Port Forwarding Still Doesn’t Work

Some ISPs make this impossible. Alternatives:

### Option 1: Mesh / Overlay VPN

- Tailscale
- ZeroTier

### Option 2: VPS Relay

- Small VPS with public IP
- Tunnel home server → VPS
- Forward traffic from VPS

### Option 3: Request Public IP

- Contact True Internet
- May cost extra, but simplest long-term fix

---

## References

- trueDDNS: [https://www.trueinternet.co.th](https://www.trueinternet.co.th)
- WireGuard: [https://www.wireguard.com](https://www.wireguard.com)
- CGNAT: [https://en.wikipedia.org/wiki/Carrier-grade_NAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT)
- RFC 6598 (CGNAT address space)

---

If you want, I can also:

- Make this shorter (blog-style)
- Turn it into a README.md
- Localize it for Thai readers
- Add diagrams / flow explanation
