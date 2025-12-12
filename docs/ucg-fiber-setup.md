# UCG-Fiber Gateway Setup for WAS-110 Monitoring

This guide covers the network configuration required when running 8311-ha-bridge from a Docker host that routes through a UniFi UCG-Fiber gateway.

## Network Topology

```
[Docker Host] --> [UCG-Fiber Gateway] --> [WAS-110 ONU]
   (your LAN)         eth6 (SFP+)         192.168.11.1
```

The WAS-110 uses a separate management subnet (192.168.11.0/24) on the SFP+ port (eth6). Your gateway needs an IP on this subnet to route traffic.

## UCG-Fiber: IP Alias Configuration (CRITICAL)

### The Problem

The WAS-110 at `192.168.11.1` is only reachable via the eth6 interface. The gateway needs an IP alias (`192.168.11.2/24`) on eth6 to route traffic to this subnet.

**However:** UniFi Network controller provisioning events can silently remove interface customizations. The standard `/data/on-boot.d/` scripts only run at boot, not after provisioning events. This can cause silent monitoring failures.

### The Solution: Cron-Based Resilience

Use a cron job that checks and restores the alias every 5 minutes.

**Execute on UCG-Fiber gateway (SSH as root):**

```bash
# Create the check script
mkdir -p /data/scripts

cat > /data/scripts/check-was110-alias.sh << 'EOF'
#!/bin/sh
# Check if eth6 has the WAS-110 management alias
# If missing, restore it and log the event
if ! ip addr show eth6 | grep -q "192.168.11.2"; then
    ip addr add 192.168.11.2/24 dev eth6 2>/dev/null
    logger -t was110-alias "Restored 192.168.11.2/24 alias to eth6"
fi
EOF

chmod +x /data/scripts/check-was110-alias.sh

# Add cron job (every 5 minutes)
(crontab -l 2>/dev/null | grep -v "check-was110-alias"; \
 echo "*/5 * * * * /data/scripts/check-was110-alias.sh") | crontab -

# Verify cron is set
crontab -l
```

### Immediate Fix (If Alias Missing Now)

```bash
ip addr add 192.168.11.2/24 dev eth6
```

### Verify Connectivity

```bash
# From gateway
ping -c 3 192.168.11.1

# Check alias is present
ip addr show eth6 | grep "192.168.11"
```

### Monitoring

Check logs for alias restoration events:

```bash
grep "was110-alias" /var/log/messages | tail -10
```

If you see frequent restorations, investigate what's triggering provisioning events in your UniFi controller.

## Docker Host: Static Route (If Needed)

If your Docker host has IP conflicts with the 192.168.11.0/24 subnet (common with Docker bridge networks), add an explicit route.

**Test first:**
```bash
ping -c 3 192.168.11.1
```

**If unreachable, add route:**
```bash
# Replace GATEWAY_IP with your UCG-Fiber's LAN IP
# Replace INTERFACE with your host's network interface
ip route add 192.168.11.1/32 via GATEWAY_IP dev INTERFACE
```

**Make persistent** (Debian/Ubuntu - add to `/etc/network/interfaces`):
```
up ip route add 192.168.11.1/32 via GATEWAY_IP dev INTERFACE
```

## SSH Key Setup (Optional)

The WAS-110 with 8311 community firmware typically allows passwordless root login. If you want to use SSH keys instead:

**Generate key on Docker host:**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/was110_key -N ""
```

**Copy to WAS-110:**
```bash
ssh-copy-id -i ~/.ssh/was110_key root@192.168.11.1
```

**Mount in Docker container** (update docker-compose.yaml):
```yaml
volumes:
  - ~/.ssh/was110_key:/root/.ssh/id_ed25519:ro
```

## Troubleshooting

### Container can't reach WAS-110

1. Check gateway alias: `ip addr show eth6 | grep 192.168.11`
2. If missing, run the immediate fix or wait for cron
3. Check Docker host routing: `ip route get 192.168.11.1`

### SSH connection refused

1. Verify WAS-110 is running: `ping 192.168.11.1`
2. Check SSH rate limiting (WAS-110 limits rapid connections)
3. Access web UI at https://192.168.11.1 to "wake" device

### Alias keeps disappearing

1. Check UniFi controller for provisioning events
2. Verify cron is running: `crontab -l`
3. Monitor logs: `grep "was110-alias" /var/log/messages`

## Files on UCG-Fiber

After setup, you should have:

```
/data/
├── scripts/
│   └── check-was110-alias.sh    # Alias check script
└── on-boot.d/
    └── 20-add-was110-alias.sh   # Optional: boot-time setup
```

Cron entry: `*/5 * * * * /data/scripts/check-was110-alias.sh`

## Why /data/?

The `/data/` directory on UniFi gateways persists across firmware updates and reboots. Always store custom scripts and configurations here.
