---
title: "Exploring & Securing Restaurant Laptop"
date: 2026-06-03
summary: "Curiosity"
showSummary: true
---

# Background

So, I currently work as a waiter to sustain my studies and my personal expenses, and for security reasons I won't be disclosing the restaurant's name. The restaurant has an HP laptop they use to check for reservations, to read incoming emails, and to check how many "too good to go" items they have along with other activities. Not to say this laptop is crucial for the job, but it's a big deal for it. Anyway, I constantly keep hearing the other staff that often use the laptop complain about how slow the laptop is, how a black window `(CMD)` just opens and closes on its own, and how the laptop starts recording or acting crazy without anyone initiating it. At first I didn't really give it much credit, and I just told them to buy a new computer, as this one is very old (i3 5th Gen, 4 GB RAM, 128 GB HDD, etc.), and maybe I can take it later to add to my home lab, hehe. One day, when the restaurant was kind of empty and I didn't have much work, I went back to the laptop, and curiosity got the best of me, so I opened the task manager, and while scrolling through the processes, I found several weird ones like `Shoopify.bat`, `Spotify.bat`, `startupppppp`, and `startuppp` and A LOT of other weird naming schemes, unfamiliar strings, and things that have no place being on a restaurant's computer, like `AutoIt` and AutoIt scripts, so I asked the restaurant to let me take it home, and they granted it to me!

# LEZZGOOO

So I took the laptop home, having in mind one thing: whatever you do, DON'T CONNECT TO THE HOME NETWORK, worried my other machines could potentially get infected. So, having the laptop at home, I started going through the task manager, and while ideal, it was using about 64% of the CPU?? So I looked at what was eating so much at it, and I found 2 "scripts" eating up exactly 50%, so I followed the file location; it was a hidden folder called ETH. We both know what that means: there is an Ethereum miner working non-stop on this machine eating up half the resources.
Okay, a quick fix: just deleted the files and folder and looked if there were any backups, and there weren't any, so after a quick reboot, the laptop was running way smoother. 
Okay, now going back to the weirdly named processes, first I wanted to do a Windows Defender scan, so I did a full scan, but what was weird and I was stupid enough to not really think about is that the scan lasted for like 3 minutes. Moving on, first, I needed to see exactly how these weird processes were launching, and standard Task Manager wasn't cutting it, so I brought in the heavy artillery **`Sysinternals Autoruns`**
I ran it as administrator, and the `shell:startup` tab looked like an absolute warzone. The startup folder was like an open reception for batch scripts, alongside `.vbs` (which at the time I didn't really know what they were) and `.lnk` files.
I started manually severing the startup hooks, but the malware gave me a run for my money. Because when I tracked the heart of everything, I stumbled on a file called `AudioSvcMangment.vbs`  that I have honestly seen a ton of times, but it looked like legit naming, so I kept ignoring it, but one of my friends pointed out the typo in the last word,  which is something a legit Windows file would almost never do. I tried deleting it, but Windows kept throwing "Access Denied." When I tried to find it in the File Explorer, it was hidden. So the malware basically changed the Access Control Lists to strip my permissions and slapped `Protected Operating System` attributes on the file. 
So, as a last resort, I went to the `cmd` and ran a force del command, which seemed to fix the error. Ouf!
After another reboot, I took another look at the task manager; it looked clean, or at least way cleaner and normal compared to how it was.

# The Blindfold

Back to the 3-minute Defender scan, I did previously ignore it, but now I am asking myself, why didn't Defender catch a somewhat obvious crypto miner and a massive AutoIt infection? Weird 
Looking into the Windows security settings, and I feel stupid for admitting this, I scrolled to the Exception list and came to find out that the malware had injected a massive list of folder exclusions directly into Defender's registry policies. It had literally blindfolded the antivirus!! 

# Finale

So after removing everything from the exception list and running another full scan, a real full scan this time, Defender woke up and started screaming.
I got spammed with severe threat notifications; Defender kept screaming at me. Most notably, the defender caught two active payloads currently loaded in the memory:
* Lumma Stealer (LummaC2): I didn't know what it was, but a quick Google search will tell you it's an infostealer that steals session cookies, crypto wallets, and saved passwords to bypass 2FA.

* Remcos RAT: Remote Access Trojan. I think most people by now know what a trojan is and how BAD and SCARY it is.

After, it was mostly easy work. I started clicked "start action" and let Defender do its magic, which took a long time, but when it eventually finished, I also installed Malwarebytes and HitmanPro to make sure the laptop was completely sanitized.
In the end, I rebooted, flew through the files and processes, and when I was somewhat confident, I connected to the restaurant's network (I went back to the restaurant) and watched the network traffic a bit; nothing was weird. I looked at the processes, and everything was good. FINALLY. I ended up setting up a firewall and lecturing the restaurant owners on security and advised them to change all their passwords and disconnect from every machine connected to the accounts.