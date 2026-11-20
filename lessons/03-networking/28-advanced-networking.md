# Lesson 28. Advanced networking: netplan, namespaces, tcpdump

## Goal
Configure networking permanently, understand the building blocks of container networks, and read raw packets.

## Permanent configuration

### Ubuntu: netplan

`/etc/netplan/01-static.yaml`:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.0.0.10/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.2, 1.1.1.1]
        search: [internal.example.com]
```

```bash
sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan try        # applies and rolls back after 120 s unless you confirm - use it on remote hosts
sudo netplan apply
```

### RHEL / NetworkManager

```bash
nmcli con show
sudo nmcli con mod eth0 ipv4.method manual ipv4.addresses 10.0.0.10/24 \
     ipv4.gateway 10.0.0.1 ipv4.dns "10.0.0.2 1.1.1.1"
sudo nmcli con up eth0
nmtui     # text UI
```

## Network namespaces — how containers get their own network

A **network namespace** is a separate network stack: own interfaces, routes, firewall rules. Every container has one. A **veth pair** is a virtual cable between two namespaces. A **bridge** is a virtual switch.

Build a mini "container network" by hand:

```bash
sudo ip netns add red
sudo ip netns add blue
sudo ip link add br0 type bridge && sudo ip link set br0 up
sudo ip addr add 10.200.0.1/24 dev br0

for ns in red blue; do
  sudo ip link add veth-$ns type veth peer name ceth-$ns
  sudo ip link set veth-$ns master br0 up
  sudo ip link set ceth-$ns netns $ns
done
sudo ip -n red  addr add 10.200.0.2/24 dev ceth-red
sudo ip -n blue addr add 10.200.0.3/24 dev ceth-blue
for ns in red blue; do
  sudo ip -n $ns link set ceth-$ns up
  sudo ip -n $ns link set lo up
  sudo ip -n $ns route add default via 10.200.0.1
done

sudo ip netns exec red ping -c2 10.200.0.3     # red talks to blue through the bridge
sudo ip netns exec red ip addr                  # only its own interfaces
```

Internet access needs forwarding + NAT on the host:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo nft add table ip nat
sudo nft add chain ip nat post '{ type nat hook postrouting priority 100; }'
sudo nft add rule ip nat post ip saddr 10.200.0.0/24 masquerade
```

This is very close to what Docker's default bridge network does (lesson 43).

Cleanup: `sudo ip netns del red; sudo ip netns del blue; sudo ip link del br0`.

## tcpdump — see the packets

```bash
sudo tcpdump -i any -n port 53                   # DNS queries
sudo tcpdump -i eth0 -n host 10.0.0.5 and port 5432
sudo tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-syn != 0'   # new connections (SYN)
sudo tcpdump -i eth0 -n -A port 80               # print payload as text (plain HTTP)
sudo tcpdump -i eth0 -w capture.pcap -c 1000     # save, open later in Wireshark
tcpdump -r capture.pcap -n
```

How to read it: `10.0.0.5.51234 > 10.0.0.10.443: Flags [S]` — SYN from client port 51234 to server 443. `[S.]` SYN-ACK, `[.]` ACK, `[P.]` data, `[F.]` close, `[R]` reset (refused).

Questions tcpdump answers in seconds:
- Do packets reach the server at all? (Firewall/security group or routing.)
- Does the server reply? (Local firewall, app, or return route.)
- Who resets the connection?

## Other useful bits

```bash
ip -s link                 # errors and drops per interface
ethtool eth0               # speed, duplex, link
sudo sysctl net.core.somaxconn net.ipv4.ip_local_port_range
ip link set eth0 mtu 1450  # MTU issues: small requests work, big ones hang
```

MTU problems (VPN, overlay networks) look like "SSH logs in but hangs on big output" or "TLS handshake hangs". Test with `ping -M do -s 1472 host`.

## Practice

1. Set a static IP with netplan using `netplan try`.
2. Build the red/blue namespaces lab, ping between them, then give them internet access with NAT.
3. Capture a DNS query and an HTTP request with tcpdump. Find the TCP handshake.
4. Connect to a closed port and find the `[R]` reset in tcpdump.

## Check yourself

- What is a veth pair and a bridge?
- Why is `netplan try` safer than `netplan apply` over SSH?
- Packets arrive on the server but the client times out. What do you check next?

## Next
[Lesson 29. Bash scripting basics](../04-automation/29-bash-basics.md)
