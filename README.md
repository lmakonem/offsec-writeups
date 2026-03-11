# Proving Grounds Walkthroughs

Personal walkthroughs for Offensive Security Proving Grounds machines and CTF events.

## Structure

```
.
├── Events/             # CTF Event Writeups
│   └── Arctic-Howl/
├── Play/
│   ├── Linux/
│   └── Windows/
└── Practice/
    ├── Linux/
    └── Windows/
```

---

## Events

### Arctic Howl - OffSec

![Expanse Surveyor](Events/Arctic-Howl/Expanse-Surveyor/images/banner.png)

| Challenge | Category | Status | Key Techniques |
|-----------|----------|--------|----------------|
| [Expanse Surveyor](Events/Arctic-Howl/Expanse-Surveyor/) | Mobile/Malware Analysis | 6/7 Completed | APK Reversing, HAR Analysis, C2 Decoding, Protobuf, GPS Tracking |

---

## Machines Completed

### Play - Linux
| Machine | Difficulty | Key Techniques |
|---------|------------|----------------|
| [Vegeta1](Play/Linux/Vegeta1.md) | Easy | Web Enum, Steganography, Morse Code, /etc/passwd Privesc |
| [Election](Play/Linux/Election.md) | Easy | Web Enum, Log File Exposure, SUID Serv-U CVE-2019-12181 |
| [My-CMSMS](Play/Linux/My-CMSMS.md) | Intermediate | MySQL Weak Creds, CMS Password Reset, UDT RCE, Base64/32 Decode, Sudo Python |

### Play - Windows
*Coming soon*

### Practice - Linux
*Coming soon*

### Practice - Windows
*Coming soon*

---

## Legend

- **Easy**: Beginner-friendly, straightforward exploitation path
- **Intermediate**: Requires some enumeration and chaining techniques
- **Hard**: Complex attack chains, custom exploitation required
