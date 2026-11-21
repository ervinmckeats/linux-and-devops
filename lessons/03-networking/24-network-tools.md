# Lesson 24. Network tools: ip, ss, ping, traceroute, dig, curl

## Goal
Diagnose any "it doesn't connect" problem layer by layer with standard tools.

## Interfaces and routes: ip

`ifconfig`, `route`, `netstat` are deprecated. Use `ip` and `ss`.

```bash
# interfaces, UP/DOWN
ip -br link

# addresses
ip -br addr

ip addr show dev eth0

# routing table
ip route

# which route/interface is used for this IP
ip route get 8.8.8.8

# ARP/neighbour table
ip neigh

# temporary (lost on reboot)
sudo ip addr add 10.0.0.5/24 dev eth0

sudo ip link set eth0 up
sudo ip route add 10.10.0.0/16 via 10.0.0.1
```

Permanent config is in netplan / NetworkManager (lesson 28).

## Sockets: ss

```bash
# listening TCP/UDP with process (run with sudo)
ss -tulpn

# active TCP connections
ss -tn state established

# connections to PostgreSQL
ss -tn dst :5432

# summary: how many in each state
ss -s

ss -tn state time-wait | wc -l

# who uses port 8080
sudo lsof -i :8080
```

Listening on `127.0.0.1:8080` = only local clients. `0.0.0.0:8080` or `[::]:8080` = all interfaces. A classic "works on the server, not from outside" bug.

## Reachability

```bash
# ICMP; many clouds and firewalls block it - no reply is not proof
ping -c 4 10.0.0.1

# path hop by hop (UDP by default)
traceroute -n example.com

# no root needed, shows MTU
tracepath example.com

# live traceroute + packet loss per hop
mtr -n example.com

# is TCP port open?
nc -zv db.internal 5432

# UDP (less reliable)
nc -zvu 10.0.0.2 53

# when nc is not installed
timeout 3 bash -c '</dev/tcp/10.0.0.2/5432' && echo open
```

## DNS: dig

```bash
# A record, full answer
dig example.com

dig +short example.com
dig example.com AAAA / MX / TXT / NS

# ask a specific resolver
dig @1.1.1.1 example.com

# follow from the root servers
dig +trace example.com

# reverse lookup
dig -x 8.8.8.8

host example.com; nslookup example.com
```

Compare your resolver vs a public one to spot stale cache or split DNS.

## HTTP: curl

```bash
# headers only
curl -I https://example.com

# full exchange, TLS handshake
curl -v https://example.com

curl -sS -o /dev/null -w '%{http_code}\n' https://example.com

# follow redirects
curl -L http://example.com

# test a vhost before DNS points to it
curl -H 'Host: site.local' http://10.0.0.5/

# same for HTTPS (SNI)
curl --resolve site.com:443:10.0.0.5 https://site.com/

curl -X POST -H 'Content-Type: application/json' -d '{"a":1}' http://localhost:8080/api

# skip TLS check (testing only)
curl -k https://self-signed.local
```

Where is the time spent?

```bash
curl -o /dev/null -s -w 'dns %{time_namelookup}  connect %{time_connect}  tls %{time_appconnect}  first-byte %{time_starttransfer}  total %{time_total}\n' https://example.com
```

## Bandwidth and capture (preview)

```bash
# on server
iperf3 -s

# on client: throughput
iperf3 -c server-ip

# who uses bandwidth now
sudo iftop -i eth0

# packets (lesson 28)
sudo tcpdump -ni eth0 port 443
```

## Bottom-up checklist

1. `ip -br link` / `ip -br addr`: interface up, address set?
2. `ip route get <dst>`: right route?
3. `ping` gateway, then destination (if ICMP is allowed).
4. `dig` / `getent hosts`: name resolves to the right IP?
5. `nc -zv host port`: port reachable? refused or timeout?
6. On the server: `ss -tlnp`: listening, on which address?
7. Firewall on both sides (lesson 26), cloud security groups.
8. `curl -v`: what does the app answer?

## Practice

1. Start `python3 -m http.server 8000 --bind 127.0.0.1`. Try to reach it from another VM. Find the reason with `ss` and fix it.
2. Measure DNS, connect, TLS and first-byte times for three sites.
3. Run `mtr` to a far-away host and find the hop where latency jumps.
4. Test a vhost with `curl -H 'Host:'` before DNS exists.

## Check yourself

- What does `0.0.0.0:80` vs `127.0.0.1:80` in `ss` mean?
- Ping fails but `curl` works. How?
- Which `curl` options show where a slow request spends its time?

## Next
[Lesson 25. SSH](25-ssh.md)
