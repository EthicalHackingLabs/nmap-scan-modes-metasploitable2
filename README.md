# 🛰️ nmap scan modes metasploitable2

> A complete walkthrough of every major Nmap scanning mode — host discovery, TCP/UDP scanning, OS detection, version fingerprinting, and aggressive scanning — run against an intentionally vulnerable VM in an isolated lab.

[![Dev.to Article](https://img.shields.io/badge/Dev.to-Read%20Article-black?logo=devdotto)](https://dev.to/almahmudkhalif/i-scanned-a-vulnerable-vm-with-every-nmap-mode-here-is-what-each-one-revealed-2eo5)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/almahmudkhalif/)

---

## 📌 Overview

Passive reconnaissance tells you what a target has exposed to the internet. Active scanning tells you what is actually running — open ports, services, versions, operating system, and more. The two phases together give you a complete picture before you write a single line of exploit code.

This project sets up an isolated lab with Kali Linux and Metasploitable2 — a deliberately vulnerable Linux VM used for security testing practice — and runs every major Nmap scanning mode against it: host discovery, TCP scanning, UDP scanning, OS detection, service version fingerprinting, and a full aggressive scan.

---

## 🛠️ Lab Setup

Two VMs on an isolated NAT network in VirtualBox — no external internet access, clean noise-free environment for packet capture.

| Machine | IP Address | Role |
|---|---|---|
| Kali Linux 2026.1 | 192.168.1.4 | Attacker |
| Metasploitable2 | 192.168.1.3 | Target |

```bash
nmap --version
# Nmap version 7.99
```

![Nmap version](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/kyi5f9b28biw28mhsj8p.png)
![Lab network setup](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/ygx36118zy5i0jn5ws8u.png)
![VM configuration](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/st7dv942613imjf44gtf.png)

---

## 🔎 Task 1 — Host Discovery

Before scanning ports, Nmap confirms the target is alive using `-sn` (no port scan).

```bash
sudo nmap -sn 192.168.1.3
```

**On a local network as root, Nmap uses ARP** — not ICMP or TCP. ARP operates below the IP layer, so firewalls blocking ICMP cannot stop it.

![ARP discovery local target](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/0jiqk96hzbni1wpxe14a.png)

The Wireshark trace confirms an ARP request/reply exchange — no ICMP, no TCP.

### Remote Target: 8.8.8.8

```bash
sudo nmap -sn 8.8.8.8
```

ARP only works within a broadcast domain, so for remote targets Nmap switches to ICMP Echo, ICMP Timestamp, TCP SYN (port 443), and TCP ACK (port 80).

![Remote host discovery](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/5b6beb5a3pz6hnz8ram6.png)

**Key insight:** ARP for local targets, ICMP+TCP for remote targets. The method changes based on network-layer reachability, not the target OS.

---

## 🔌 Task 2 — TCP Port Scanning

```bash
sudo nmap -sT 192.168.1.3   # Connect scan — full TCP handshake
sudo nmap -sS 192.168.1.3   # SYN scan — half-open, faster, quieter
```

![TCP connect scan](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/shg6qnn6a8ke9qbuoxjh.png)
![TCP SYN scan](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/s2gn4xjwhzzlur221sqv.png)

**Result: 23 ports open, 977 closed.**

```
PORT      STATE  SERVICE
21/tcp    open   ftp
22/tcp    open   ssh
23/tcp    open   telnet
25/tcp    open   smtp
53/tcp    open   domain
80/tcp    open   http
111/tcp   open   rpcbind
139/tcp   open   netbios-ssn
445/tcp   open   microsoft-ds
512/tcp   open   exec
513/tcp   open   login
514/tcp   open   shell
1099/tcp  open   rmiregistry
1524/tcp  open   ingreslock
2049/tcp  open   nfs
2121/tcp  open   ccproxy-ftp
3306/tcp  open   mysql
5432/tcp  open   postgresql
5900/tcp  open   vnc
6000/tcp  open   X11
6667/tcp  open   irc
8009/tcp  open   ajp13
8180/tcp  open   unknown
```

Telnet sends credentials in plaintext, r-services (512/513/514) carry weak authentication, port 1524 is an intentional backdoor shell, and databases plus VNC are reachable with zero firewall restriction.

---

## 📶 Task 3 — UDP Port Scanning

```bash
sudo nmap -sU -sV -T4 --top-ports=100 192.168.1.3
```

![UDP scan results](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/4yh67b5xqot8il5vdh6k.png)

**4 UDP ports confirmed open:**

| Port | Service | Version |
|---|---|---|
| 53/udp | domain | ISC BIND 9.4.2 |
| 111/udp | rpcbind | 2 (RPC #100000) |
| 137/udp | netbios-ns | Microsoft Windows netbios-ns |
| 2049/udp | nfs | 2-4 (RPC #100003) |

Without `-sV`, most of these would show as `open|filtered` — the version probe is what confirms them as definitively open.

---

## 🧬 Task 4 — OS Detection

```bash
sudo nmap -O 192.168.1.3
```

![OS detection output](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/qvv37wii7hv3p18o0zfh.png)

| Field | Value |
|---|---|
| Device type | general purpose |
| OS CPE | `cpe:/o:linux:linux_kernel:2.6` |
| OS details | Linux 2.6.9 – 2.6.33 |

Confirmed inside the VM:

```bash
uname -r
# 2.6.24-16-server
```

![Kernel version confirmation](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/cnwqe8fdua92489sluwp.png)

The real kernel (`2.6.24`) sits squarely within Nmap's predicted range — fingerprinting is probabilistic but accurate enough to target version-specific exploits.

---

## 🏷️ Task 5 — Version and Service Scanning

```bash
sudo nmap -sV 192.168.1.3
```

![Version scan output](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/3at1bxj58pcpdyh7bfgd.png)

| Port | Service | Version |
|---|---|---|
| 21/tcp | ftp | vsftpd 2.3.4 |
| 22/tcp | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 53/tcp | domain | ISC BIND 9.4.2 |
| 80/tcp | http | Apache httpd 2.2.8 (Ubuntu) DAV/2 |
| 139/tcp | netbios-ssn | Samba smbd 3.X – 4.X |
| 1524/tcp | bindshell | Metasploitable root shell |
| 3306/tcp | mysql | MySQL 5.0.51a-3ubuntu5 |
| 5432/tcp | postgresql | PostgreSQL DB 8.3.0 – 8.3.7 |
| 5900/tcp | vnc | VNC (protocol 3.3) |
| 8180/tcp | http | Apache Tomcat/Coyote JSP engine 1.1 |

**vsftpd 2.3.4** contains a known backdoor — appending `:)` to a username during FTP login opens a root shell on port 6200.

---

## 💥 Task 6 — Complete Aggressive Scan

```bash
sudo nmap -A 192.168.1.3
```

![Aggressive scan part 1](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/bwpe3cbv2l42g2anotqs.png)
![Aggressive scan part 2](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/26434bzuia7dwfs1f2i9.png)
![Aggressive scan part 3](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/vaw5gvm47ctpwzym63je.png)
![Aggressive scan part 4](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/rvkoc8en7ilhxyb3374r.png)
![Aggressive scan part 5](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/9c8i7egcsbz9vq288cve.png)

**Key findings:**

- FTP: anonymous login allowed — no credentials required
- VNC: no authentication — Security Type None
- SMTP: PIPELINING, VRFY enabled (username enumeration possible)
- IRC: UnrealIRCd on `irc.Metasploitable.LAN`
- Samba: computer name `metasploitable`, domain `localdomain`
- MySQL: Protocol 10, version 5.0.51a
- 2048-bit RSA SSH host key fingerprint captured

With a root shell on 1524, a backdoored vsftpd on 21, unauthenticated VNC, and anonymous FTP, this single host offers multiple independent paths to full compromise.

---

## 📊 The Complete Nmap Scanning Sequence

| Flag | Scan Type | How It Works | When to Use |
|---|---|---|---|
| `-sn` | Host discovery | ARP (local) or ICMP+TCP (remote) | Confirming targets before scanning |
| `-sT` | TCP connect | Full three-way handshake | When you do not have root |
| `-sS` | TCP SYN | Half-open (SYN → SYN-ACK → RST) | Default with root — faster, quieter |
| `-sU` | UDP | Protocol probes + ICMP port unreachable | Finding UDP services often missed |
| `-O` | OS detection | TCP/IP stack fingerprinting | Targeting kernel-specific exploits |
| `-sV` | Version scan | Banner grabbing + service probes | Finding exact software versions |
| `-A` | Aggressive | All of the above + NSE scripts + traceroute | Deep dive on specific confirmed targets |

---

## ⚠️ Common Mistakes

| Mistake | What Happens |
|---|---|
| Running without `sudo` | No ARP on local networks, weaker host detection |
| Skipping `-sV` on UDP | Most UDP ports show as `open\|filtered` — unusable |
| Default timing on remote UDP | Scan can take hours |
| Running `-A` against full subnets | Takes forever, generates huge noise |
| Trusting port numbers alone | Port 1524 shows as `ingreslock` without version scanning |

---

## 📖 Full Walkthrough

Read the complete step-by-step article on Dev.to:

👉 [I Scanned a Vulnerable VM with Every Nmap Mode — Here's What Each One Revealed](https://dev.to/almahmudkhalif/i-scanned-a-vulnerable-vm-with-every-nmap-mode-here-is-what-each-one-revealed-2eo5)

---

## 🌐 Connect With Me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/almahmudkhalif/)
[![Dev.to](https://img.shields.io/badge/Dev.to-Articles-black?logo=devdotto)](https://dev.to/almahmudkhalif/)
