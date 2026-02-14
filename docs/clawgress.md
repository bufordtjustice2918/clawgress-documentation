# Clawgress Policy Engine

Clawgress uses a single `policy.json` source of truth to manage DNS RPZ (via bind9) and egress firewall rules (via nftables).

## Install (ISO)

1. Download the latest ISO from the build artifacts.
2. Boot the ISO in your hypervisor (VMware/VirtualBox/QEMU).
3. Login and install to disk (standard VyOS install flow).
4. Reboot into the installed system.

> Note: Clawgress uses bind9 for DNS/RPZ; dnsmasq is not used for DNS forwarding.

## CLI Usage

The `clawgress` command provides operational control over the policy.

### Show Current Policy
```bash
clawgress show
```

### Import and Apply Policy
```bash
clawgress import --policy /path/to/my-policy.json
```

### Apply Existing Policy
```bash
clawgress apply
```

### Check Status
```bash
clawgress status
```

## Minimal policy.json
```json
{
  "version": 1,
  "allow": {
    "domains": ["api.openai.com"],
    "ips": ["1.2.3.4/32"],
    "ports": [53, 80, 443]
  },
  "labels": {
    "api.openai.com": "llm-provider"
  }
}
```

## API Usage

### Status
**Endpoint:** `GET /clawgress/health`

Returns bind9 + nftables status and recent deny stats.

The Clawgress REST API (running on port 8080 by default) allows remote policy updates.

### Update Policy
**Endpoint:** `POST /clawgress/policy`

**Payload:**
```json
{
  "key": "your-api-key",
  "policy": {
    "version": 1,
    "allow": {
      "domains": ["api.openai.com"],
      "ips": ["1.2.3.4/32"],
      "ports": [443]
    },
    "labels": {
      "api.openai.com": "llm-provider"
    }
  },
  "apply": true
}
```

## Architecture

1. **Policy Input:** JSON submitted via CLI or API.
2. **DNS RPZ:** `clawgress-policy-apply` generates bind9 RPZ zones.
3. **Firewall:** `clawgress-firewall-apply` generates nftables rules in the `inet clawgress` table.
4. **Forced DNS:** All outbound traffic on port 53 (UDP/TCP) is redirected to the local bind9 resolver.

## DNS Configuration

Clawgress uses **bind9** for DNS resolution and RPZ (Response Policy Zones). This differs from standard VyOS which uses dnsmasq for DNS forwarding.

- **DNS Service:** bind9 handles all DNS queries and RPZ policy enforcement
- **DHCP Service:** Remains under `service dhcp-server` (dnsmasq is not used for DNS)
- **Forced DNS:** Outbound DNS traffic (port 53) is redirected to the local bind9 resolver via nftables rules
- **Logging:** DNS queries are logged to syslog with policy labels for denied domains

To configure DHCP without DNS forwarding:
```bash
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 dns-server 192.168.1.1
```

This directs DHCP clients to use the Clawgress bind9 resolver instead of external DNS.
