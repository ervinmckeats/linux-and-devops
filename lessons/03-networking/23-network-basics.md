# Lesson 23. Network basics: IP, ports, TCP/UDP, DNS

## Goal
Have a mental model of how a request travels, so network problems become a checklist, not magic.

## Layers you actually use

| Layer | What | Examples | Tools |
|---|---|---|---|
| Link (L2) | frames on a local network, MAC addresses | Ethernet, Wi-Fi, ARP | `ip link`, `ip neigh` |
| Network (L3) | packets between networks, IP addresses, routing | IPv4, IPv6, ICMP | `ip addr`, `ip route`, `ping`, `traceroute` |
| Transport (L4) | connections between ports | TCP, UDP | `ss`, `nc` |
| Application (L7) | the protocol of the app | HTTP, DNS, SSH, TLS | `curl`, `dig`, `openssl` |

Debug **bottom-up**: link up? IP set? route exists? DNS resolves? port open? app answers?

## IP addresses and subnets

`192.168.1.10/24` = address + prefix length. `/24` means the first 24 bits are the network: `192.168.1.0`–`192.168.1.255`, 254 usable hosts.

| Prefix | Hosts | Typical use |
|---|---|---|
| /32 | 1 | single host in firewall rules |
| /24 | 254 | a LAN or a cloud subnet |
| /16 | 65 534 | a cloud VPC |

Private ranges (not routed on the internet): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Also `127.0.0.0/8` (loopback) and `169.254.0.0/16` (link-local, cloud metadata `169.254.169.254`).

**NAT** lets many private hosts share one public IP. **Default gateway** is the router for "everything else". IPv6 addresses look like `2001:db8::1/64`; same ideas, no NAT needed.

## Ports, TCP and UDP

A **port** (0–65535) identifies an application on a host. A connection is the 5-tuple: protocol, src IP, src port, dst IP, dst port.

Well-known ports: 22 SSH, 25 SMTP, 53 DNS, 80 HTTP, 443 HTTPS, 3306 MySQL, 5432 PostgreSQL, 6379 Redis, 9090 Prometheus. Ports below 1024 need root (or `CAP_NET_BIND_SERVICE`).

| | TCP | UDP |
|---|---|---|
| Connection | yes, 3-way handshake | no |
| Delivery | reliable, ordered, retransmits | best effort |
| Use | HTTP, SSH, databases | DNS, video, metrics (StatsD), QUIC |

TCP handshake: `SYN →`, `← SYN-ACK`, `ACK →`. Closing goes through `FIN` and `TIME_WAIT`.

What the client sees tells you a lot:
- **Connection refused** — host reachable, nothing listens on that port (or a firewall sends reject).
- **Timeout** — packets dropped: firewall, wrong route, host down.
- **No route to host** — routing problem or ICMP reject.

## DNS

Turns names into addresses. Resolution path: `/etc/hosts` → local resolver (`systemd-resolved` at `127.0.0.53`) → recursive resolver (ISP, 1.1.1.1, VPC DNS) → root → TLD → authoritative server.

| Record | Holds |
|---|---|
| A / AAAA | IPv4 / IPv6 address |
| CNAME | alias to another name |
| MX | mail servers |
| TXT | text: SPF, domain verification |
| NS | authoritative servers of a zone |
| SRV | service host and port |
| PTR | reverse: IP → name |

**TTL** = how long resolvers cache an answer. Lower TTL to 60–300 s a day **before** a migration.

Order of lookups is set in `/etc/nsswitch.conf` (`hosts: files dns`). Resolver config: `/etc/resolv.conf`, `resolvectl status`.

"It's always DNS" is a joke because it is so often true. Check it early.

## Commands

```bash
ip -br addr             # my addresses
ip route                # routing table, default gateway
cat /etc/resolv.conf; resolvectl status
getent hosts example.com    # resolve the same way apps do (hosts file + DNS)
ipcalc 10.20.0.0/20         # subnet math (package ipcalc)
```

## Practice

1. Find your IP, prefix, gateway and DNS server.
2. How many hosts in `/20`? Which addresses are in `10.0.16.0/20`?
3. Add `1.2.3.4 test.local` to `/etc/hosts` and resolve it with `getent` and with `dig` — why do the answers differ?
4. Connect to a closed port on localhost and to a filtered IP (e.g. `10.255.255.1:80`). Compare "refused" and "timeout".

## Check yourself

- Connection refused vs timeout: what does each one tell you?
- Why is `dig` not always the same as what your app sees?
- What should you do with the TTL before moving a site to a new IP?

## Next
[Lesson 24. Network tools](24-network-tools.md)
