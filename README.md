# Heimdall: A Modern Homebrew Gateway with Debian 13

This guide walks you through transforming a clean install of **Debian 13 (Trixie)** into a high-performance, secure home router and gateway.

## Features
* **Bufferbloat Mitigation:** Optimized with BBR + FQ network congestion control for symmetrical fiber outside cgnat.
* **Privacy-First DNS:** DNSmasq chained to Stubby for upstream DNS-over-TLS (DoT) with built-in malware blocking.
* **Hardware Longevity:** Zram memory compression and minimized systemd journal logging to protect SSD lifespans.
* **Secure VPN Gateway:** High-performance WireGuard server pre-configured with QR code profile generation for mobile devices.

---
## Quick info
* WAN = enp2s0
* LAN = enp3s0
* WIREGUARD = wg0

---

## Step 1: Base System Optimization

### 1. Update & Dependencies
Log in as `root` or use `sudo` to update the core system and pull down the required routing and networking tools:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install htop nfs-kernel-server rpcbind openssh-server \
dnsmasq wireguard qrencode stubby \
bind9-dnsutils zram-tools -y
```

### 2. SSD Write Minimization & Zram
Set up compressed RAM layout using `lz4` compression to lower SSD wear cycles.

```bash
sudo nano /etc/default/zramswap
```
Add the following layout configuration:
```ini
ALGO=lz4
PERCENT=33
PRIORITY=100
```

Create a custom sysctl optimization profile to reduce swap aggression:
```bash
sudo nano /etc/sysctl.d/99-zram-opt.conf
```
```ini
vm.page-cluster = 0
vm.swappiness = 150
```

Load the active optimizations:
```bash
sudo sysctl --load=/etc/sysctl.d/99-zram-opt.conf
```

### 3. Volatile System Logging (Optional)
Move standard system runtime logs into temporary volatile memory space (`tmpfs`) to further prevent disk wear:

```bash
sudo nano /etc/systemd/journald.conf
```
Uncomment or append the following keys under the `[Journal]` header:
```ini
[Journal]
Storage=volatile
RuntimeMaxUse=64M
```
Restart the logging daemon:
```bash
sudo systemctl restart systemd-journald
```

### 4. Enable Storage TRIM
Verify your underlying SSD trim timers are active to keep long-term operational storage performance crisp:

```bash
sudo systemctl status fstrim.timer
```
If not enabled yet then enable it
```bash
sudo systemctl enable --now fstrim.timer
```

---

## Step 2: Interface & Routing Topology

### 1. Identify Network Interface Cards (NICs)
Locate your exact physical WAN and LAN interface designations:
```bash
ip link
```

### 2. Configure Local Networking
Map out static LAN allocations and standard dynamic WAN links:
```bash
sudo nano /etc/network/interfaces
```
```ini
# WAN Interface
auto enp2s0
iface enp2s0 inet dhcp

# LAN Interface
auto enp3s0
iface enp3s0 inet static
    address 192.168.10.1
    netmask 255.255.255.0
```
Cycle the network service stacks to activate parameters:
```bash
sudo systemctl restart networking
```

### 3. Kernel IP Forwarding
Instruct the Debian kernel to route traffic between interfaces dynamically:
```bash
sudo nano /etc/sysctl.d/99-ipforward.conf
```
```ini
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 0
net.ipv6.conf.default.forwarding = 0
```

### 4. Congestion Control & Fiber Latency Tweaks
Use the correct congestion scheduler for your enviroment / setup
alongside targeted buffer parameters:

```bash
sudo nano /etc/sysctl.d/99-tcp-congestion.conf
```
Symmetric fiber with its own ipv4 address, no cgnat
```ini
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```
Cable/Coax/DSL
```ini
net.core.default_qdisc = cake
net.ipv4.tcp_congestion_control = cubic
```
CGNAT / Shared Residential
```ini
net.core.default_qdisc = fq_codel
net.ipv4.tcp_congestion_control = cubic
```

Optimize internal queue depth allocations:
```bash
sudo nano /etc/sysctl.d/99-router-latency.conf
```
```ini
net.ipv4.tcp_low_latency = 1
net.core.netdev_max_backlog = 5000

# Fiber-optimized memory spaces (Up to 32MB TCP window sizes)
net.core.rmem_max = 33554432
net.core.wmem_max = 33554432
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432
```
Commit all core network routing states to runtime:
```bash
sudo sysctl --load=/etc/sysctl.d/99-ipforward.conf
sudo sysctl --load=/etc/sysctl.d/99-tcp-congestion.conf
sudo sysctl --load=/etc/sysctl.d/99-router-latency.conf
```

---

## Step 3: Firewall Construction (IPv4 & IPv6 Templates)

> [!WARNING]
> Double-check that your active LAN interface matches `enp3s0` exactly in the configuration blocks below before executing rules to avoid getting permanently locked out of SSH.

### 1. Current Standard: nftables (Recommended)
```bash
nano nftables.conf
```
```config
#!/usr/sbin/nft -f

# 1. Clear any existing rules for a clean, atomic reload
flush ruleset

table inet filter {
    # Define your IP sets natively (Shared cleanly across filter and NAT chains)
    set offline {
        type ipv4_addr
        elements = { 192.168.10.201, 192.168.10.202, 192.168.10.203, 192.168.10.204 }
    }

    set dns_bypass {
        type ipv4_addr
        elements = { 192.168.10.91, 192.168.10.92, 192.168.10.93, 192.168.10.99 }
    }

    chain input {
        type filter hook input priority filter; policy drop;

        # Allow established and related traffic (Stateful inspection)
        ct state established,related accept

        # Allow Loopback (Localhost) - handles both IPv4 and IPv6 loopback
        iif "lo" accept

        # --- Router-Only IPv6 ICMP Handling ---
        # Allows necessary IPv6 ICMP types for the router's own WAN interface
        ip6 nexthdr icmpv6 accept

        # --- IPv4 Specific Rules ---
        # Allow incoming Ping (ICMP Echo) only from the Local LAN
        iif "enp3s0" icmp type echo-request accept

        # Allow LAN and VPN clients access to local router services (DNS, DHCP, SSH)
        iif "enp3s0" accept
        iifname "wg0" accept # FIX: Prevents boot errors when wg0 is missing

        # Allow WAN incoming connection for WireGuard handshake packets
        iif "enp2s0" udp dport 51820 accept

        # Explicitly drop any remaining incoming traffic hitting the WAN interface (enp2s0)
        iif "enp2s0" drop
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # Allow established and related traffic (Stateful inspection)
        ct state established,related accept

        # OFFLINE SET BLOCKS (Checked immediately after established connections)
        # Block offline set from sending data out to WAN (enp2s0)
        iif "enp3s0" oif "enp2s0" ip saddr @offline drop

        # Block unsolicited internet traffic (enp2s0) from hitting offline set
        iif "enp2s0" oif "enp3s0" ip daddr @offline drop

        # Block DoT so standard devices fall back to Port 53 (where they get hijacked to dnsmasq)
        iif "enp3s0" ip saddr != @dns_bypass tcp dport 853 drop
        iif "enp3s0" ip saddr != @dns_bypass udp dport 853 drop

        # AUTOMATIC PORT FORWARDING FILTER
        # Securely allows any traffic that matches a DNAT rule in the prerouting chain.
        ct status dnat accept

        # IPv4 Forwarding & Outbound Routing
        # Route internal traffic out to the internet
        iif "enp3s0" oif "enp2s0" accept
        iifname "wg0" oif "enp2s0" accept # FIX: Prevents boot errors

        # Inter-LAN / Inter-VPN Local Forwarding
        iif "enp3s0" oifname "wg0" accept # FIX: Prevents boot errors
        iifname "wg0" oif "enp3s0" accept # FIX: Prevents boot errors

        # Dynamically fix MTU/MSS issues for WireGuard (TCPMSS clamping)
        tcp flags & (syn|rst) == syn tcp option maxseg size set rt mtu

        # IPv6 Drop and Safe Reject Rules
        ## 1. Global Drops (Handles all unhandled traffic, including any rogue IPv6 transit)
        iif "enp2s0" drop

        ## 2. Reject internal IPv6 traffic trying to go out to WAN so devices instantly fail over to IPv4
        iif "enp3s0" oif "enp2s0" meta nfproto ipv6 reject with icmpv6 type admin-prohibited
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }

    # NAT CHAINS (Integrated safely inside the family-independent 'inet' table)
    chain prerouting {
        type nat hook prerouting priority filter; policy accept;

        # FORCE ROUTER DNS (HIJACK PORT 53)
        iif "enp3s0" ip saddr @dns_bypass accept

        # For ALL other devices, redirect their Port 53 requests to local dnsmasq
        iif "enp3s0" udp dport 53 redirect to :53
        iif "enp3s0" tcp dport 53 redirect to :53

        # PORT FORWARDING DNAT RULES

        # EXAMPLE (192.168.10.70)
        ## FTP
        #iif "enp2s0" tcp dport { 990, 40000-40100 } dnat ip to 192.168.10.70
        ## MINECRAFT
        #iif "enp2s0" tcp dport 19134 dnat ip to 192.168.10.70
        #iif "enp2s0" udp dport { 19134, 19145-19154 } dnat ip to 192.168.10.70
        ## SUNSHINE
        #iif "enp2s0" tcp dport { 47984, 47989, 48010 } dnat ip to 192.168.10.70
        #iif "enp2s0" udp dport { 47998-48000, 48002, 48010 } dnat ip to 192.168.10.70
    }

    chain postrouting {
        type nat hook postrouting priority filter; policy accept;

        # NAT/Masquerade local IPv4 traffic into the WAN interface IP
        oif "enp2s0" masquerade
    }
}

```
### 2. Quick info about config
IPV6 only allowed for wan and router, blocked for lan
Drop everything incoming by default unless rule is defined like port forwarding
offline set is meant for older devices that need no internet or internet trying to talk to them,
like older access points, managed switches, old avr and such.
dns_bypass is for devices that might need other dns resolvers like a gaming console if you use a massive ad blocking file that blocks microsoft servers that handle achievement tracking and such.
DNS, port 853 is dropped unless in dns_bypass set, port 53 is hijacked and pointed to dnsmasq.
Portforwarding example is left to show how to forward ports, ranges and protocol to ip / host.

### 3. apply default firewall template
#### check config, make sure wan and lan eth matches yours
```bash
sudo nft --check --file nftables.conf
```
#### copy and apply strict permissions just incase
```bash
sudo cp nftables.conf /etc/nftables.conf
sudo chown root:root /etc/nftables.conf
sudo chmod 600 /etc/nftables.conf
```
#### enable service, start and status
```bash
sudo systemctl enable nftables.service
sudo systemctl restart nftables.service
sudo systemctl status nftables.service
```
#### now lets check if the firewall rules are active
```bash
sudo nft list ruleset
```
##### should like similar to below
<details>
table inet filter {
	set offline {
		type ipv4_addr
		elements = { 192.168.10.201, 192.168.10.202, 192.168.10.203, 192.168.10.204 }
	}

	set dns_bypass {
		type ipv4_addr
		elements = { 192.168.10.91, 192.168.10.92, 192.168.10.93, 192.168.10.99 }
	}

	chain input {
		type filter hook input priority filter; policy drop;
		ct state established,related accept
		iif "lo" accept
		ip6 nexthdr ipv6-icmp accept
		iif "enp3s0" icmp type echo-request accept
		iif "enp3s0" accept
		iifname "wg0" accept
		iif "enp2s0" udp dport 51820 accept
		iif "enp2s0" drop
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state established,related accept
		iif "enp3s0" oif "enp2s0" ip saddr @offline drop
		iif "enp2s0" oif "enp3s0" ip daddr @offline drop
		iif "enp3s0" ip saddr != @dns_bypass tcp dport 853 drop
		iif "enp3s0" ip saddr != @dns_bypass udp dport 853 drop
		ct status dnat accept
		iif "enp3s0" oif "enp2s0" accept
		iifname "wg0" oif "enp2s0" accept
		iif "enp3s0" oifname "wg0" accept
		iifname "wg0" oif "enp3s0" accept
		tcp flags & (syn | rst) == syn tcp option maxseg size set rt mtu
		iif "enp2s0" drop
		iif "enp3s0" oif "enp2s0" reject with icmpv6 admin-prohibited
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}

	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
		iif "enp3s0" ip saddr @dns_bypass accept
		iif "enp3s0" udp dport 53 redirect to :53
		iif "enp3s0" tcp dport 53 redirect to :53
	}

	chain postrouting {
		type nat hook postrouting priority filter; policy accept;
		oif "enp2s0" masquerade
	}
}

</details>


<details>
    <summary>Click here to view the legacy IPTables setup (Deprecated)</summary>

```bash
sudo apt install iptables-persistent -y
```
### 2.
offline

### 1. IPv4 Filter Set (`rules.v4`)
```bash
nano rules.v4
```
```text
*mangle
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
COMMIT
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -i enp3s0 -j ACCEPT
-A INPUT -i wg0 -j ACCEPT
-A INPUT -i enp2s0 -p udp -m udp --dport 51820 -j ACCEPT
-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i enp3s0 -o enp2s0 -j ACCEPT
-A FORWARD -i wg0 -o enp2s0 -j ACCEPT
-A FORWARD -i enp3s0 -o wg0 -j ACCEPT
-A FORWARD -i wg0 -o enp3s0 -j ACCEPT
-A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
COMMIT
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o enp2s0 -j MASQUERADE
COMMIT
```

### 2. IPv6 Filter Set (`rules.v6`)
```bash
nano rules.v6
```
```text
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -i enp3s0 -p ipv6-icmp -j ACCEPT
-A INPUT -i wg0 -p ipv6-icmp -j ACCEPT
-A INPUT -i enp2s0 -p ipv6-icmp -m icmp6 --icmpv6-type 2 -j ACCEPT
-A INPUT -i enp2s0 -p ipv6-icmp -m icmp6 --icmpv6-type 3 -j ACCEPT
-A INPUT -i enp2s0 -p ipv6-icmp -m icmp6 --icmpv6-type 4 -j ACCEPT
-A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 128 -j ACCEPT
-A INPUT -i enp2s0 -j DROP
-A FORWARD -i enp3s0 -p ipv6-icmp -j ACCEPT
-A FORWARD -i enp3s0 -o enp2s0 -j REJECT --reject-with icmp6-adm-prohibited
-A FORWARD -i wg0 -o enp2s0 -j REJECT --reject-with icmp6-adm-prohibited
-A FORWARD -i enp2s0 -j DROP
COMMIT
```

### 3. Commit Rules System-Wide
Restore the local target files straight into operating memory, then use `netfilter-persistent` to auto-commit them permanently across reboots:
```bash
sudo iptables-restore < rules.v4
sudo ip6tables-restore < rules.v6
sudo netfilter-persistent save
```
</details>
---

## Step 4: Encrypted DNS Stacking (Stubby + DNSmasq)

### 1. Deploy Stubby DoT Configurations
Overwrite your default configuration layout with an isolated localhost-bound DNS-over-TLS query profile utilizing secure Cloudflare upstream targets:

```bash
sudo nano /etc/stubby/stubby.yml
```
```yaml
resolution_type: GETDNS_RESOLUTION_STUB
dns_transport_list:
  - GETDNS_TRANSPORT_TLS
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
edns_client_subnet_private : 1
round_robin_upstreams: 0
idle_timeout: 10000
timeout: 1500

listen_addresses:
  - 127.0.0.1@5353
  - 0::1@5353

upstream_recursive_servers:

# PRIMARY: Quad9 Malware Filtering & DNSSEC
  - address_data: 9.9.9.9
    tls_auth_name: "dns.quad9.net"
  - address_data: 149.112.112.112
    tls_auth_name: "dns.quad9.net"

# FALLBACK: Cloudflare (Malware Blocking Only)
  - address_data: 1.1.1.2
    tls_auth_name: "security.cloudflare-dns.com"
  - address_data: 1.0.0.2
    tls_auth_name: "security.cloudflare-dns.com"
```
```bash
sudo systemctl enable --now stubby
dig @127.0.0.1 -p 5353 google.com
```

### 2. Configure DNSmasq Local Resolver
Tie local client caching directly into the background cryptographic Stubby listener loop:
Copy your own adblocking hosts file to /etc/hosts_adblocking,
if you dont have one use touch command to set a dummy file that can be replaced later

```bash
sudo nano /etc/dnsmasq.conf
```
```ini
except-interface=enp2s0
listen-address=127.0.0.1,192.168.10.1,10.8.0.1
domain=asgard
local=/asgard/
expand-hosts
domain-needed
bogus-priv
no-resolv

#here DHCP range is set from 120 to 180, thats more than enough for me, static ips i use outside that range
dhcp-range=192.168.10.120,192.168.10.180,255.255.255.0,12h
dhcp-option=option:router,192.168.10.1
dhcp-option=option:dns-server,192.168.10.1
dhcp-authoritative

addn-hosts=/etc/hosts_adblocking
cache-size=30000

server=127.0.0.1#5353
filter-AAAA
```
```bash
sudo touch /etc/hosts_adblocking
sudo systemctl enable --now dnsmasq
```
#### DNSmasq DHCP and static addresses and hostnames
```bash
sudo nano /etc/dnsmasq.d/static_hosts.conf
```
```config
dhcp-host=a1:a1:a1:1b:1b:ab,192.168.10.91,XBOXONES
dhcp-host=00:00:01:00:00:00,192.168.10.92,XBOXONE
dhcp-host=a0:a0:a0:1a:1a:ab,192.168.10.93,XBOXSERIESS
dhcp-host=5c:b9:00:00:00:00,192.168.10.199,HPPRINTER

```
Replace my dummy mac addresses with your own, but use : as in how my dummy macs are presented,
then the local ip address you want it to always have, and lastly hostname to make it easy to see and connect too.

Now we need to make sure same hostnames are in our local hosts file that handle local lan adressing
```bash
sudo nano /etc/hosts
```
Copy below or replicate with your own static set ip's and hostnames like in my examples
Local hosts is what interest us, just go below the written lines in your hosts file and create a #local hosts
if it doesn't exists and write the IP and HOSTNAME beneath it.
```config
# Local hosts
192.168.10.91 XBOXONES
192.168.10.92 XBOXONE
192.168.10.93 XBOXSERIESS
192.168.10.199 HPPRINTER
```
---


## Step 5: WireGuard Endpoint Integration

### 1. Key Generation
```bash
sudo mkdir -p /etc/wireguard && sudo -i
cd /etc/wireguard
umask 077
cd /root

# Server
wg genkey | tee server_private.key | wg pubkey > server_public.key

# Clients
wg genkey | tee phone_private.key | wg pubkey > phone_public.key
wg genkey | tee tablet_private.key | wg pubkey > tablet_public.key
wg genkey | tee laptop_private.key | wg pubkey > laptop_public.key
```

### 2. Server Interface Assignment (`wg0.conf`)
```bash
nano /etc/wireguard/wg0.conf
```
```ini
[Interface]
PrivateKey = <PASTE_CONTENTS_OF_server_private.key>
Address = 10.8.0.1/24
ListenPort = 51820
[Peer]
PublicKey = <PASTE_CONTENTS_OF_phone_public.key>
AllowedIPs = 10.8.0.2/32
[Peer]
PublicKey = <PASTE_CONTENTS_OF_tablet_public.key>
AllowedIPs = 10.8.0.3/32
[Peer]
PublicKey = <PASTE_CONTENTS_OF_laptop_public.key>
AllowedIPs = 10.8.0.4/32
```
```bash
exit
sudo systemctl enable --now wg-quick@wg0
```
3. Mobile Device Onboarding (QR Deploy)
Construct local peer file structures matching individual hardware profiles (e.g., phone.conf):
```ini
[Interface]
PrivateKey = <PASTE_CONTENTS_OF_phone_private.key>
Address = 10.8.0.2/24
DNS = 10.8.0.1
[Peer]
PublicKey = <PASTE_CONTENTS_OF_server_public.key>
Endpoint = <PUBLIC_WAN_IP_OR_DDNS>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```
Get keys
```bash
sudo -i
cat server_public.key
cat server_private.key
cat phone_public.key
cat phone_private.key
cat tablet_public.key
cat tablet_private.key
cat laptop_public.key
cat laptop_private.key
exit
```
Render an administrative, inline terminal deployment link profile for rapid QR capture over the air:
```bash
qrencode -t ansiutf8 < phone.conf
```
---

## 📄 License

This project is open-source and licensed under the **MIT License**. Feel free to use, modify, and distribute it as you see fit. See the accompanying `LICENSE` file for full legal details.

