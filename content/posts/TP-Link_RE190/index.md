---
title: "A Deep Dive into the TP-Link RE190"
date: 2026-02-17
summary: "Second Embedded Project"
showSummary: true
---

# Opening up the TP-Link RE190

I bought a TP-Link RE190 WiFi extender to see if I could break into it, maybe find a backdoor, a vulnerability or something interesting.

At first, the device was a total ghost. I connected to it but `nmap` scans against the gateway IP (`172.25.135.254`) showed nothing but filtered ports and a Cisco MAC address 

I had to get clever. I used `ip neigh` to inspect the ARP table, finally finding the device by its unique TP-Link MAC OUI (`50:3d:d1...`). By pinging the domain `tplinkrepeater.net`, I discovered it had pulled a dynamic IP (`172.25.132.113`) from the upstream DHCP server, correcting my initial assumption that it would be at the default address


## Web Probing

I started by force-browsing for hidden diagnostic pages and found `/web/modules/system/diagnostic.html`. I tried to inject a payload into the NTP Time Server field to spawn a root shell:
`0.pool.ntp.org; /usr/sbin/telnetd -p 23 -l /bin/sh`

The web interface blocked me with an "Invalid Input" error. This was just client-side JavaScript validation and I bypassed it by opening the browser's Developer Tools and manually deleting the `pattern` and `maxlength` attributes from the HTML input tag!

I hit "Apply," but the router's internal command parser—**TPOS 2.0**—refused to execute the shell command because it doesn't understand Linux chaining like `;`

## Fw Dumping

Since the software was a shutdown door,I turned to the board.

**Hardware:**
* **Chipset:** MediaTek MT7628 (MIPS architecture)
* **Flash:** 4MB (EN25QH32B)

I used a **CH341a programmer** with an SOIC8 clip to dump the firmware out of the **4MB EN25QH32B flash chip**. 
I identified the layout:
* **CONFIG partition:** `0x003fc000` (8KB)
* **FIRMWARE partition:** `0x0000d000`

## Digital Autopsy (Ghidra & Grep)

The real work began with the raw `dump.bin`. `binwalk` revealed a chaotic mix of false positives and one massive 5MB file at offset `0xD080`. I tried to carve out the filesystem at offset `1589059`, but I was blocked by a proprietary, non-standard compression header.

I loaded the 5MB `decompressed.bin` into Ghidra as a raw **MIPS32 Little Endian** binary

The technical hurdles were brutal:
1.  **Base Address:** I mapped the kernel to `0x80000000` to resolve the jump tables
2.  **MIPS Delay Slots:** I hit "Flow Override" errors and had to manually force disassembly on the bytes in the delay slots to reveal the valid code packed behind them
3.  **Credential Logic:**  found internal debugging strings like `wlan_key` and `ipcam_auth_password`. I traced the error message "invalid PSK password" back to a function that handled the authentication logic.

## The key


**1. The Hardcoded Backdoor (CWE-321)**
I discovered a valid **1024-bit RSA Private Key** hardcoded directly in the firmware. I extracted the block matching `-----BEGIN RSA PRIVATE KEY-----` and used OpenSSL to prove it was a mathematically valid key.

```bash
root@archlinux:~/tplink_dump# grep -A 5 "BEGIN RSA" decompressed.bin 
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDCklE2... [REDACTED] ...

-----END RSA PRIVATE KEY-----
``` 
**2. The Symmetric "Kill Shot" (DES Key)**
While the RSA key compromised the HTTPS layer, I needed the symmetric key used for data encryption. A deep dive into the decompressed web assets revealed a  comment from a developer dating back to 2007.

Found in ./2478FE/decompressed.bin:

var iterations = key.length > 8 ? 3 : 1; //changed by Paul 16/6/2007 to use Triple DES for 9+ byte keys

Following this thread, I located the specific authentication key hardcoded in the binary:

authKey WaQ7xbhc9TefbwK

This 15-character key (WaQ7xbhc9TefbwK) triggers the 3DES logic, effectively acting as the master key for the device's client-side encryption.

## Potential exploits

**Vector A: The Passive Wiretap**
Because I extracted the symmetric authKey (WaQ7xbhc9TefbwK), an attacker does not need to guess the admin password.

    Intercept: An attacker on the local network captures the HTTP login traffic (Wireshark).

    Decrypt: Using the hardcoded 3DES key found in the firmware, they can decrypt the login payload offline.

    Result: They recover the cleartext admin password without ever sending a packet to the router.

**Vector B: Persistent Root**
I found strings related to ipssh.auth.none.allowed, proving the device has a hidden SSH server.

    Download: An attacker downloads the config.bin backup.

    Decrypt: Using the custom DES key, they decrypt the file.

    Modify: They toggle the SSH flag to true and inject their own authorized keys.

    Upload: They restore the modified backup file to gain a permanent root shell.
