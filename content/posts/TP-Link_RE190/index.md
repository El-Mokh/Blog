---
title: "A Deep Dive into the TP-Link RE190"
date: 2026-02-17
summary: "Second Embedded Project"
showSummary: true
---

# The Digital Autopsy: Breaking the TP-Link RE190

I bought a TP-Link RE190 WiFi extender to see if I could break into it, maybe find a backdoor, a vulnerability or something interesting. This wasn't just about plugging in wires, it was a digital autopsy of a proprietary, monolithic OS.

## The Ghost on the Network

At first, the device was a total ghost. I connected to its "Open Setup Network", but `nmap` scans against the gateway IP (`172.25.135.254`) showed nothing but filtered ports and a Cisco MAC address. I realized the extender was acting as a , passing traffic through without showing itself.

I had to get clever. I used `ip neigh` to inspect the ARP table, finally de-cloaking the device by its unique TP-Link MAC OUI (`50:3d:d1...`). By pinging the magic domain `tplinkrepeater.net`, I discovered it had pulled a dynamic IP (`172.25.132.113`) from the upstream DHCP server, correcting my initial assumption that it would be at the default address.

*(The ARP table revealing the hidden TP-Link device)*

## Phase 2: Web Probing & The JavaScript Bypass

I started by force-browsing for hidden diagnostic pages and found `/web/modules/system/diagnostic.html`. I tried to inject a payload into the NTP Time Server field to spawn a root shell:
`0.pool.ntp.org; /usr/sbin/telnetd -p 23 -l /bin/sh`

The web interface blocked me with an "Invalid Input" error. This was just client-side JavaScript validation. I bypassed it by opening the browser's Developer Tools and manually deleting the `pattern` and `maxlength` attributes from the HTML input tag.

I hit "Apply," but the router's internal command parser—**TPOS 2.0**—refused to execute the shell command because it doesn't understand standard Linux chaining like `;`. The software door was shut.

## Phase 3: Hardware Extraction (CH341a)

Since the software was "lobotomized," I went straight for the silicon. I cracked the case open to inspect the board.

**Vital Stats:**
* **Chipset:** MediaTek MT7628 (MIPS architecture)
* **Flash:** 4MB (EN25QH32B)

Instead of messing with the live device immediately, I wanted a "dead" copy of the brain. I used a **CH341a programmer** with an SOIC8 clip to perform a raw hardware dump of the **4MB EN25QH32B flash chip**. This allowed me to extract the firmware without the device even being powered on.

I identified the layout immediately:
* **CONFIG partition:** `0x003fc000` (8KB)
* **FIRMWARE partition:** `0x0000d000`

## Phase 4: Digital Autopsy (Ghidra & Grep)

The real work began with the raw `dump.bin`. `binwalk` revealed a chaotic mix of false positives and one massive 5MB file at offset `0xD080`. I tried to carve out the filesystem at offset `1589059`, but I was blocked by a proprietary, non-standard compression header.

I loaded the 5MB `decompressed.bin` into Ghidra as a raw **MIPS32 Little Endian** binary. This was a monolithic RTOS—no filesystem, just raw machine code.

The technical hurdles were brutal:
1.  **Base Address:** I mapped the kernel to `0x80000000` to resolve the jump tables.
2.  **MIPS Delay Slots:** I hit "Flow Override" errors and had to manually force disassembly (pressing 'D') on the bytes in the delay slots to reveal the valid code packed behind them.
3.  **Credential Logic:** Using `grep -E`, I found internal debugging strings like `wlan_key` and `ipcam_auth_password`. I traced the error message "invalid PSK password" back to function `FUN_803b1748`, which handled the authentication logic.

## Phase 5: The Loot (The Skeleton Key)

By hunting for raw text patterns in the dump, I found the smoking gun. The firmware was a security nightmare.

**1. The Hardcoded Backdoor (CWE-321)**
I discovered a valid **1024-bit RSA Private Key** hardcoded directly in the firmware. I extracted the block matching `-----BEGIN RSA PRIVATE KEY-----` and used OpenSSL to prove it was a mathematically valid key.

```bash
root@archlinux:~/tplink_dump# grep -A 5 "BEGIN RSA" decompressed.bin 
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDCklE2... [REDACTED] ...
... [REDACTED] ...
-----END RSA PRIVATE KEY-----
``` 
**2. The Symmetric "Kill Shot" (DES Key)**
While the RSA key compromised the HTTPS layer, I needed the symmetric key used for data encryption. A deep dive into the decompressed web assets revealed a critical comment from a developer named "Paul" dating back to 2007.

// Found in ./2478FE/decompressed.bin
var iterations = key.length > 8 ? 3 : 1; //changed by Paul 16/6/2007 to use Triple DES for 9+ byte keys

Following this thread, I located the specific authentication key hardcoded in the binary:
// Found in ./2AB2C6/decompressed.bin
authKey WaQ7xbhc9TefbwK

This 15-character key (WaQ7xbhc9TefbwK) triggers the 3DES logic, effectively acting as the master key for the device's client-side encryption.

## Phase 6: The Live Breach (Bus Pirate 5)

With the secrets extracted and the map in hand, I switched to the Bus Pirate v5 to interact with the live device. As an Arch Linux user on my TUXEDO, I added my user to the uucp group (sudo usermod -aG uucp $USER) and used tio with --flow none for a stable serial terminal.

I established a "Split Power" configuration: my Raspberry Pi provided the 3.3V, while the Bus Pirate handled the data lines. I caught the U-Boot 1.1.3 interrupt and was dropped into a root-level console: UART>, verifying that I had total control over the hardware.

(The "Frankenstein" Rig: Bus Pirate v5 + Raspberry Pi power injection)

## Phase 7: The Kill Chain (How to Exploit)

Finding these keys isn't just a "theoretical" issue. It enables a complete compromise of the device through two specific vectors:

**Vector A: The Passive Wiretap (No Auth Required)**
Because I extracted the symmetric authKey (WaQ7xbhc9TefbwK), an attacker does not need to guess the admin password.

    Intercept: An attacker on the local network captures the HTTP login traffic (Wireshark).

    Decrypt: Using the hardcoded 3DES key found in the firmware, they can decrypt the login payload offline.

    Result: They recover the cleartext admin password without ever sending a packet to the router.

**Vector B: Persistent Root (The SSH Backdoor)**
I found strings related to ipssh.auth.none.allowed, proving the device has a hidden SSH server.

    Download: An attacker downloads the config.bin backup.

    Decrypt: Using the custom DES key, they decrypt the file.

    Modify: They toggle the SSH flag to true and inject their own authorized keys.

    Upload: They restore the modified backup file to gain a permanent root shell.

## Conclusion

The security of this device relies entirely on "Security by Obscurity." By putting the private key in the firmware, the manufacturer effectively taped the key to the lock. I even did a factory reset to see if it would generate a new key—it stayed exactly the same.

By hardcoding the encryption keys, TP-Link has reduced the security of this device to a single static string that is now public knowledge. Any RE190 deployed in the wild is vulnerable to immediate, passive credential theft.

I have successfully mapped the path to full compromise: from unboxing a CH341a programmer to identifying a global, model-specific skeleton key.