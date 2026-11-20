# Lesson 26. Firewalls: ufw, firewalld, nftables

## Goal
Allow only the traffic you need, without locking yourself out.

## Theory

The Linux kernel filters packets with **netfilter**. Tools on top of it:

| Tool | Where | Notes |
|---|---|---|
| `nftables` (`nft`) | modern kernel interface | replaces iptables |
| `iptables` | legacy, still everywhere in docs | on new systems it is often a wrapper over nftables |
| `ufw` | Ubuntu | simple front-end |
| `firewalld` | RHEL, Fedora | zones, front-end |

Use **one** front-end per host. Mixing them creates rules nobody understands.

Principles:
- **Default deny incoming**, allow outgoing, open only what you need.
- Firewalls are **stateful**: allow a new incoming connection on port 443, and the replies are allowed automatically (`ESTABLISHED,RELATED`).
- Cloud VMs have a second firewall outside the VM: **security groups** / network ACLs. Both must allow the traffic.

## Do not lock yourself out

1. Keep a second SSH session open while changing rules.
2. Allow SSH **before** enabling the firewall.
3. Have console access (cloud serial console) as a fallback.
4. For risky changes schedule a rollback: `sudo at now + 5 minutes <<< 'ufw disable'`, then cancel it with `atrm` if all is well.

## ufw (Ubuntu)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80,443/tcp
sudo ufw allow from 10.0.0.0/24 to any port 5432 proto tcp
sudo ufw limit 22/tcp
sudo ufw enable
sudo ufw status numbered
sudo ufw delete 3
sudo ufw app list
```

- `sudo ufw allow OpenSSH`: or `sudo ufw allow 22/tcp`
- `sudo ufw limit 22/tcp`: rate-limit brute force

Docker warning: Docker writes its own iptables rules, and published container ports (`-p 8080:80`) **bypass ufw**. Bind to localhost (`-p 127.0.0.1:8080:80`) or use the `DOCKER-USER` chain.

## firewalld (RHEL)

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --zone=trusted --add-source=10.0.0.0/24
sudo firewall-cmd --reload
```

Without `--permanent` a rule is runtime-only and is lost on reload. A common trick: test without `--permanent`, then `--runtime-to-permanent`.

## nftables

`/etc/nftables.conf`:

```
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept
    ct state invalid drop
    iif lo accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept
    tcp dport 22 ct state new limit rate 10/minute accept
    tcp dport { 80, 443 } accept
    ip saddr 10.0.0.0/24 tcp dport 5432 accept
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output  { type filter hook output priority 0; policy accept; }
}
```

```bash
sudo nft -c -f /etc/nftables.conf
sudo nft -f /etc/nftables.conf
sudo nft list ruleset
sudo systemctl enable nftables
```

- `sudo nft -c -f /etc/nftables.conf`: check syntax
- `sudo nft -f /etc/nftables.conf`: apply

## iptables (for reading old docs)

```bash
sudo iptables -L -n -v --line-numbers
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables-save > rules.v4
```

- `sudo iptables-save > rules.v4`: persist with iptables-persistent

## Check from outside

```bash
nmap -Pn -p 1-1000 your-server
nc -zv your-server 5432
```

- `nmap -Pn -p 1-1000 your-server`: from ANOTHER machine, only on hosts you own

## Practice

1. On a VM: default deny, allow SSH and 80/443, enable ufw, check with `nmap` from another VM.
2. Allow PostgreSQL (5432) only from one IP.
3. Run a Docker container with `-p 8080:80` and prove it bypasses ufw. Fix it.
4. Write the same policy in nftables on a second VM.
5. Practice the `at` rollback trick.

## Check yourself

- Why do you not need a rule for reply packets?
- What two firewalls does a cloud VM have?
- Why can a Docker port be open even though ufw denies it?

## Next
[Lesson 27. HTTP, TLS, nginx](27-http-tls-nginx.md)
