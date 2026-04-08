# vpn - Simple WireGuard User Manager

A Bash script to manage WireGuard users and access control on a single WireGuard server with a flexible tag-based system.

Features:

- Initialize a new WireGuard server
- Add/remove VPN users with tag-based access control
- Tag-based IP organization with directory support
- Flexible access rules (full network, single IP, multiple IPs, tag references, directory wildcards)
- Automatic iptables ACL management using ipset
- User and tag explanation commands

---

## Before Starting

Make sure the following are true:

1. **OS / Privileges**
   - Ubuntu/Debian-based server
   - You have `sudo` access (root or sudoer)

2. **Network**
   - The server has Internet access (outbound HTTPS)
   - Your router/firewall forwards **UDP 51820** to this server's IP
   - If you are behind NAT, the forwarded public IP is the address you expect clients to reach

3. **Not already running another WireGuard setup on `wg0`**
   - If you already have `/etc/wireguard/wg0.conf` for another purpose, review this script carefully before running `init`

4. **Packages**
   - The `init` command will install: `wireguard`, `wireguard-tools`, `curl`, `iptables`, `ipset`, `iptables-persistent`
   - If WireGuard is already installed, you can use `start` to only install the script

---

## Quick Start

1. Clone the repo and make script executable:

   ```bash
   git clone <your_repo_url> vpn-manager
   cd vpn-manager
   chmod +x vpn
   ```

2. Run full init (installs WireGuard + dependencies + iptables ACL + installs script):

   ```bash
   ./vpn init
   ```

   This will:

   - Install `wireguard`, `wireguard-tools`, `curl`, `iptables`, `ipset`, `iptables-persistent`
   - Create directory structure:
     - `/home/wireguard/clients` - Client configuration files
     - `/home/wireguard/keys` - Private/public keys
     - `/home/wireguard/users.txt` - User registry
     - `/home/wireguard/tags` - Tag definitions
   - Create iptables chain `WG_ACCESS` and hook it into `FORWARD -i wg0`
   - Install script to `/usr/local/bin/vpn`

3. Check usage:

   ```bash
   vpn guide
   ```

---

## Commands

### 1. Initialize System

```bash
./vpn init
```

- Use this **once** on a fresh server
- Installs all dependencies (WireGuard, iptables, ipset)
- Creates directory structure
- Sets up iptables ACL chain
- Installs script to `/usr/local/bin/vpn`

### 2. Install Script Only

```bash
./vpn start
```

- Only copies the script to `/usr/local/bin/vpn`
- Use this if WireGuard is already installed and configured

### 3. Add User

```bash
vpn add <username> <target>
```

The `target` parameter supports multiple formats:

- **`full`** - Access to entire LAN subnet (192.168.219.0/24)
  ```bash
  vpn add alice full
  ```

- **Single IP** - Access to one specific IP
  ```bash
  vpn add bob 192.168.0.10
  ```

- **Multiple IPs** - Comma-separated list
  ```bash
  vpn add charlie 192.168.0.10,192.168.0.11
  ```

- **Tag reference** - Use a predefined tag
  ```bash
  vpn add david example/db
  ```

- **Directory wildcard** - All tags in a directory
  ```bash
  vpn add eve example/
  ```

Each `add` command will:
- Append a `[Peer]` block to `/etc/wireguard/wg0.conf`
- Assign a VPN IP `10.0.0.X/32`
- Generate keys in `/home/wireguard/keys`
- Log entry in `/home/wireguard/users.txt`
- Create iptables rules using ipset for efficient filtering
- Apply rules immediately (no restart needed)

---

### 4. Remove User

```bash
vpn remove <username>
```

This will:
- Remove the peer from running `wg0`
- Delete keys and configuration
- Remove log entry from `users.txt`
- Remove associated iptables/ipset rules

---

### 5. List Users

```bash
vpn list
```

Shows all users with:
- Username
- VPN IP (10.0.0.X)
- Access rule definition
- Creation date

---

### 6. Tag Management

Create tags to organize IP addresses:

```bash
# Add a tag
vpn tag add <ip> <tag_name>

# Example: Create a database server tag
vpn tag add 192.168.0.100 production/db

# Remove a tag
vpn tag remove <tag_name>
```

Tags support directory structure:
- `production/db`
- `production/web`
- `development/api`

Use directory wildcards when adding users: `vpn add user production/`

---

### 7. Explain Command

Get detailed information about users, tags, or directories:

```bash
# Explain a user's access
vpn explain <username>

# Show what IP a tag points to
vpn explain <tag_name>

# List all tags in a directory
vpn explain <directory/>
```

---

### 8. Show Guide

```bash
vpn guide
```

Displays usage guide and available commands

---

## Access Control Model

This script uses a layered security approach:

### WireGuard Layer
- Server `wg0.conf` `[Peer]` blocks use `AllowedIPs = 10.0.0.X/32` to identify each client
- This does **not** control LAN access directly

### iptables + ipset Layer
- All access control is enforced via the `WG_ACCESS` iptables chain
- For `full` access: Direct iptables rule allowing entire LAN subnet
- For specific IPs: Uses ipset for efficient multi-IP matching
  - Each user gets a dedicated ipset: `user_10_0_0_X`
  - Tag/directory references are resolved to IPs and added to the set
  - iptables rule matches against the ipset
- Final DROP rule ensures deny-by-default

### Security Benefits
- Even if a client modifies their config file, the server controls actual access
- ipset provides O(1) lookup for large IP lists
- Tag system allows centralized IP management
- Directory structure enables logical grouping

---

## Example Workflows

### Example 1: Database Access

```bash
# Create tags for database servers
vpn tag add 192.168.0.100 production/db-primary
vpn tag add 192.168.0.101 production/db-replica

# Add a DBA with access to both
vpn add dba production/

# Add a developer with limited access
vpn add dev1 production/db-replica
```

### Example 2: Multi-Environment Access

```bash
# Set up environment tags
vpn tag add 192.168.0.10 dev/api
vpn tag add 192.168.0.11 dev/web
vpn tag add 192.168.0.20 staging/api
vpn tag add 192.168.0.21 staging/web

# Developer gets all dev environment
vpn add developer dev/

# QA gets dev + staging
vpn add qa dev/,staging/
```

### Example 3: Mixed Access

```bash
# Specific IPs + tag reference + directory
vpn add admin 192.168.0.1,production/db,monitoring/
```

---

## Notes

- **VPN subnet**: `10.0.0.0/24` (configurable in script)
- **Default LAN subnet**: `192.168.219.0/24` (change `add_acl_rules` function if needed)
- **Interface**: `wg0` only
- **Tag storage**: `/home/wireguard/tags/` with directory support
- Always review and test the script before production use
