# Lab Network Topology

This document describes the network architecture of the home SOC lab: how
the isolated attack/target network is built, how it's wired for traffic
mirroring, and how that mirroring survives restarts. Written as a
reference for reproducing or extending the setup.

## Overview

The lab has two distinct networks on the Proxmox host (SV1):

- **Home network** (`192.168.8.0/24`, via `vmbr0`) — used for management
  access to each VM (web consoles, SSH). Reachable from any device on the
  home LAN.
- **Isolated lab network** (`10.10.10.0/24`, via `vmbr1`) — a Proxmox-only
  virtual bridge with no physical NIC attached and no DHCP server. This is
  where the actual attacker/target/monitoring traffic lives, fully
  separated from the home network and the internet.

This split mirrors how Security Onion itself is built (a management
interface plus a separate, IP-less monitor interface), and was extended to
the attacker and target VMs as they were added.

## Hosts

| VM              | VMID | Role                | Home network (`vmbr0`) | Lab network (`vmbr1`)      |
|------------------|------|---------------------|--------------------------|------------------------------|
| Proxmox host     | —    | Hypervisor          | `192.168.8.10`           | —                             |
| security-onion   | 101  | SIEM / NSM, sniffing| `192.168.8.12`           | `ens19` — IP-less, monitor only |
| kali-attacker    | 103  | Attacker             | `192.168.8.13`           | `10.10.10.10`                |
| metasploitable   | 104  | Vulnerable target    | *(none — isolated only)* | `10.10.10.20`                |

All home-network addresses are fixed via DHCP reservations in the router
(GL-MT6000), rather than static IPs configured inside each OS — this
matches how the rest of the home lab (Proxmox itself, other servers) is
already managed, keeping one consistent convention across the whole
environment.

Metasploitable intentionally has no home-network interface at all —
per the project's own documentation, it should never be exposed to an
untrusted network, and it has no legitimate need for management access
beyond the isolated segment's console.

## Why isolated instead of monitoring the real home network

Two approaches were considered for what Security Onion should actually
monitor:

1. Tap into the real home network (requires a managed switch with port
   mirroring/SPAN support, or a physical network tap; captures real
   household device traffic).
2. Build a fully isolated, Proxmox-only virtual network for attacker/
   target traffic (no physical network involvement at all).

Went with option 2. It's safer (no risk of misconfiguring the real home
network or capturing/logging household devices' actual traffic), matches
how most guided SOC lab material assumes the environment is built (e.g.
TryHackMe's SOC Level 1 path), and the underlying analyst skills — SIEM
use, detection writing, alert triage — transfer identically whether the
traffic is real household traffic or lab-generated traffic between VMs.
A home-network tap could be added later as a separate phase if wanted.

## The `vmbr1` bridge

- Type: Linux Bridge
- IPv4/CIDR: none (IP-less — it only carries traffic between the VMs
  attached to it, doesn't need its own address)
- No physical NIC attached
- MTU: 9000 (jumbo frames) — required to match Security Onion's own
  monitor-interface configuration; see the MTU note below.
- Autostart: enabled

## Why MTU 9000 matters here

Security Onion's Standalone deployment configures its monitor interface
for jumbo frames (MTU 9000) by default — this is Security Onion's own
documented Proxmox guidance, not a choice made for this lab specifically.
`vmbr1` and every VM's NIC attached to it need to support at least that
MTU, or Security Onion's monitor-interface bonding will fail (see
`troubleshooting-detection-pipeline.md` for the full story of what that
failure looks like and how it was diagnosed). Proxmox's default MTU for a
new bridge is 1500, so this needs to be set explicitly — it does not
happen automatically.

## Traffic mirroring

A standard Linux bridge (like `vmbr1`) forwards unicast traffic directly
between the two ports involved in a conversation, based on MAC address
learning — it does **not** flood that traffic to every port the way a
hub would. This means Security Onion's monitor interface, even in
promiscuous mode, will not see traffic between Kali and Metasploitable
by default — promiscuous mode only affects what a port sees when traffic
actually arrives there, and on a normal bridge, traffic between two other
ports simply never arrives.

To fix this, explicit port mirroring is configured using Linux `tc`
(traffic control), mirroring both attacker and target VMs' traffic to
Security Onion's tap interface:

```
tc qdisc add dev <source-tap> handle ffff: ingress
tc filter add dev <source-tap> parent ffff: protocol all u32 match u32 0 0 action mirred egress mirror dev <security-onion-tap>
```

Where the tap interface names follow Proxmox's `tap<VMID>i<index>`
convention:

| VM             | Tap interface |
|-----------------|----------------|
| kali-attacker    | `tap103i0`     |
| metasploitable   | `tap104i0`     |
| security-onion   | `tap101i1` (the monitor NIC specifically, `net1`) |

## Making mirroring persistent

Tap interfaces are destroyed and recreated by Proxmox every time a VM
stops and starts, which means `tc` rules applied by hand don't survive a
VM restart. This is solved with a Proxmox **hookscript** —
`/var/lib/vz/snippets/mirror-hook.sh` — attached to both the Kali and
Metasploitable VMs, which reapplies the mirroring rule automatically on
every `post-start` event:

```bash
#!/bin/bash
VMID="$1"
PHASE="$2"

if [ "$PHASE" != "post-start" ]; then
    exit 0
fi

if [ "$VMID" != "103" ] && [ "$VMID" != "104" ]; then
    exit 0
fi

TARGET_TAP="tap101i1"
SOURCE_TAP="tap${VMID}i0"

sleep 2  # let the tap interface fully settle

tc qdisc add dev "$SOURCE_TAP" handle ffff: ingress 2>/dev/null
tc filter add dev "$SOURCE_TAP" parent ffff: protocol all u32 match u32 0 0 action mirred egress mirror dev "$TARGET_TAP" 2>/dev/null

exit 0
```

Attached via:
```
qm set 103 --hookscript local:snippets/mirror-hook.sh
qm set 104 --hookscript local:snippets/mirror-hook.sh
```

This has been verified working after both individual VM restarts and a
full Proxmox host reboot. One caveat found during testing: a VM that
stays running through a host reboot (rather than itself stopping and
starting) will **not** trigger the hookscript, since no `post-start`
event occurs — its mirroring rule can be lost without the usual
re-triggering mechanism kicking in. If a host reboot happens and a lab VM
was left running through it, check `tc filter show dev <tap> parent
ffff:` on that VM's tap interface and cycle it (`qm stop` / `qm start`)
if the rule is missing.

## Quick verification

To confirm the whole pipeline is working after any change:

```
# On Security Onion, watch for traffic while triggering some from Kali:
sudo tcpdump -i bond0 -n host 10.10.10.20

# From Kali:
ping -c 4 10.10.10.20
```

A working setup shows the full ARP resolution and ICMP exchange in the
tcpdump output. From there, Security Onion's Hunt page
(`source.ip:10.10.10.10 OR destination.ip:10.10.10.20`) should show
corresponding Zeek connection logs, and an actual scan (e.g. `nmap -sV
10.10.10.20`) should generate real Suricata alerts.
