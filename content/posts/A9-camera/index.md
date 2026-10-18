---
title: "Reverse Engineering a Generic IoT Camera"
date: 2026-01-05
summary: "First Embedded Project"

showSummary: true
---

I bought a generic IoT camera to see if I could break into it. I hooked up my Pi to the camera’s UART interface.

* **Chip:** Beken BK7252N
* **OS:** RT-Thread OS v3.1.0
* **Kernel Address:** `0x10000`

## Fake Shell

To shut up the log spam so I could actually type, I set up a mobile hotspot using the stolen credentials the camera was screaming for(SSID & PASSWORD). Instantly, the camera connected, the noise died down, and I was dropped into a clean shell prompt: `msh />`.

I hit TAB to get the command list and saw promising tools like `t_readFlashAddr`, `w_r16d8Read` (a direct register reader?), and `rfcali_show_data`.

```text
[I/FAL] Fal(V0.4.0)success
[I/OTA] RT-Thread OTA package(V0.2.4) initialize success.
go os_addr(0x10000)..........

 \ | /
- RT -     Thread Operating System
 / | \     3.1.0 build Aug 10 2024
 2006 - 2018 Copyright by rt-thread team 
 
[FUNC]rwnxl_init
[FUNC]calibration_main
xtal in flash is:30
lwIP-2.0.2 initialized!

codeBuildTime:2024年 08月 10日 星期六 17:35:47 CST 
com.naxclow.Camera_v0-1.1.20 
DevType (NAXC_2M_48P_B_A9_versimsh />[0]202408101735 

# --- CREDENTIAL LEAK ---
_wifi_easyjoin: ssid:NAXCLOW bssid:00:00:00:00:00:00 key:34974033A
# ----------------------------------------

rtc_videoMalloc 0x905768 
normal_connect11
wpa_supplicant_req_scan
PSKC: ssid NAXCLOW
wpa_supplicant_connect
wpa_driver_associate

# --- WIFI CONNECTED ---
[WLAN_MGNT]wlan sta connected event callback 
sta_ip_start
configuring interface mlan (with DHCP client)
IP UP: 172.20.10.11 

# --- ROOT SHELL ACCESS ---
msh /> 
msh /> help
msh: help: command not found
msh /> 
RT-Thread shell commands:
rxsens           - do_rx_sens
txevm            - do_tx_evm
rfcali_show_data - rfcali show data
t_readDevice     - t_readDevice....
t_readMACAddr    - t_readMACAddr....
t_readFlashAddr  - t_readFlashAddr
reboot           - asdsadsad
w_r16d8Read      - w_r16d8Read
w_r16d8Write     - w_r16d8Write
msh />
msh />
```

For a second I thought I won, I found my way into the firmware and it was easy and striaghtforward! But no, the commands were just not working.
## The Hardware Wall

Thought of dumping the firmware through the flash chip but unfortunately it was a SiP. 

## The Pivot: Targetting the Bootloader

After a quick Google search, I was introduced to `bk7231tools`, a custom python utility designed to exploit the serial protocol of Beken chips

The theory was simple: The chip listens for a specific "handshake" byte sequence for milliseconds when it first wakes up and If it hears it, it pauses the boot process and enters a raw data transfer mode. If it misses it, it boots the OS

```text
el-mokh@raspberrypi:~$ bk7231tools read_flash -d /dev/serial0 firmware_2M_backup.bin --length 2097152 --baudrate 115200
```

It took multiple attempts but text started scrolling. I was pulling data but in my excitement and repeated handling of the board, I put too much tension on the wiring.

![Ripped Pad Damage](/img/BK7252.png)
*I ripped the pad during the process, but I secured the dump first.*


## Fw analysis
Tried running some commands fronm the github tool but it just told me the dump was corrupted. Bad news.And i couldn't retrieve another one because the pad had already been ripped.
Also I found hardcoded wifi credentials in the firmware. The camera is hardcoded to  look for this specific wifi credentials with a spwcific password.
So i just kept running regex on the binary hoping to find something interesting.
Here is what I found inside the dump:

**1. Hardcoded Backdoor**
```text
SSID: NAXCLOW
PASS: 34974033A
```

**2. Cloud Identity**

the Cloud Authentication Token and Device ID
```text
Device ID: 0800d1099...... [REDACTED]
Token:     qefc90031649913745bc02df69...... [REDACTED]
```

**3. Cloud API Endpoints**

```text
Endpoint: v720.naxclow.com
API:/app/api/ApiAlertRecord # And different other ones
```