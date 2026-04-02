# vpn - Simple WireGuard User Manager

This is a small Bash script to manage WireGuard users and access control on a single WireGuard server.

It can:

- Initialize a new WireGuard server (optional)
- Add/remove VPN users
- Generate client `.conf` files
- Restrict each user’s access to your LAN using iptables (full / single / multi)
- Restart WireGuard
- Show user list and get client configs

---

## Before Starting

Make sure the following are true:

1. **OS / Privileges**
   - Ubuntu/Debian-based server.
   - You have `sudo` access (root or sudoer).

2. **Network**
   - The server has Internet access (outbound HTTPS).
   - Your router/firewall forwards **UDP 51820** to this server’s IP.
   - If you are behind NAT, the forwarded public IP is the address you expect clients to reach.

3. **Not already running another WireGuard setup on `wg0`**
   - If you already have `/etc/wireguard/wg0.conf` for another purpose, review this script carefully before running `init`. It will create or modify `wg0.conf`.

4. **Packages**
   - For `init`:
     - `apt` must be available (Debian/Ubuntu).
   - For manual usage (`start` or only `vpn add`):
     - `wireguard`, `wireguard-tools`, `iptables`, `curl` should be installed already.

---

## Quick Start (Fresh Server, v1)

1. Clone the repo and make script executable:

   ```bash
   git clone <your_repo_url> vpn-manager
   cd vpn-manager
   chmod +x vpn
   ```

2. Run full init (installs WireGuard + basic wg0.conf + iptables ACL + installs script):

   ```bash
   ./vpn init
   ```

   This will:

   - Install `wireguard`, `wireguard-tools`, `curl`.
   - Create `/etc/wireguard/wg0.conf` with:
     - Interface `wg0`
     - Address `10.0.0.1/24`
     - ListenPort `51820`
     - Basic iptables NAT rules.
   - Create `/home/wireguard/clients`, `/home/wireguard/keys`, `/home/wireguard/users.txt`.
   - Create iptables chain `WG_ACCESS` and hook it into `FORWARD -i wg0`.
   - Install script to `/usr/local/bin/vpn`.

3. Check usage:

   ```bash
   vpn guide
   ```

---

## Commands

### 1. `init` (only in v1)

```bash
./vpn init
```

- Use this **once** on a fresh server.
- Installs dependencies and sets up:
  - WireGuard (`wg0`)
  - iptables ACL chain
  - `/usr/local/bin/vpn`

### 2. `start` (only in v1)

```bash
./vpn start
```

- Only copies the script to `/usr/local/bin/vpn` (no apt, no wg0.conf).
- Use this if WireGuard is already installed and configured.

### 3. Add user

```bash
vpn add <username> <full|single|multi> [target]
```

- `full`  
  - User can access:
    - `10.0.0.0/24` (VPN subnet)
    - `192.168.219.0/24` (your home LAN, example)
  - Example:
    ```bash
    vpn add alice full
    ```

- `single`  
  - User can access only one LAN IP.
  - Example:
    ```bash
    vpn add bob single 192.168.219.100
    ```

- `multi`  
  - User can access multiple LAN IPs (comma-separated).
  - Example:
    ```bash
    vpn add charlie multi 192.168.219.100,192.168.219.101
    ```

Each `add` will:

- Append a `[Peer]` block to `/etc/wireguard/wg0.conf`.
- Assign a VPN IP `10.0.0.X/32`.
- Create keys in `/home/wireguard/keys`.
- Create client config in `/home/wireguard/clients/<username>.conf`.
- Log entry in `/home/wireguard/users.txt`.
- Add iptables rules in `WG_ACCESS` based on `TYPE` and `TARGET`.

> After adding users, **restart WireGuard**:

```bash
vpn restart
```

---

### 4. Remove user

```bash
vpn remove <username>
```

This will:

- Remove the peer from the running `wg0`.
- Remove the peer block from `/etc/wireguard/wg0.conf`.
- Delete keys and client config.
- Remove the log line from `users.txt`.
- Remove iptables rules for that user’s VPN IP.

Then:

```bash
vpn restart
```

---

### 5. List users

```bash
vpn list
```

Prints:

- Username
- VPN IP (AllowedIPs in server config)
- TYPE (full/single/multi)
- TARGET (allowed LAN IPs, if any)

---

### 6. Restart WireGuard

```bash
vpn restart
```

- Runs `systemctl restart wg-quick@wg0`.
- Shows first few lines of status.

---

### 7. Get client config

```bash
vpn get <username>
```

- Prints `/home/wireguard/clients/<username>.conf`.

You can send this file (or copy-paste content) to your user and import it into the WireGuard client app.

---

## Access Control Model

- Server-side `[Peer]` in `wg0.conf`:

  - `AllowedIPs = 10.0.0.X/32` (identifies each client).
  - Does **not** control LAN access directly.

- Client-side config:

  - `AllowedIPs = 10.0.0.0/24,192.168.219.0/24`
  - Controls routing only (which destinations go through the tunnel).

- Actual access control:

  - Enforced with iptables chain `WG_ACCESS`:
    - `full`: `-s 10.0.0.X/32 -d 192.168.219.0/24 -j ACCEPT` then `DROP`
    - `single`: `-s 10.0.0.X/32 -d 192.168.219.Y/32 -j ACCEPT` then `DROP`
    - `multi`: multiple `-d IP/32 -j ACCEPT` then `DROP`

So even if a client edits their config, the server still decides what they can reach.

---

## Notes

- This script is opinionated and tailored for:
  - Single WireGuard interface named `wg0`.
  - VPN subnet `10.0.0.0/24`.
  - Example LAN subnet `192.168.219.0/24` (change in script if needed).
- Always review the script before using it on production systems.