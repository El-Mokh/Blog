---
title: "Level 6"
---

* Login : ssh leviathan6@leviathan.labs.overthewire.org -p 2223

In the home directory you'll find an `ELF` called `leviathan6`. When you run it :

```bash
leviathan6@leviathan:~$ ./leviathan6 
usage: ./leviathan6 <4 digit code>
leviathan6@leviathan:~$ ./leviathan6 0000
Wrong
leviathan6@leviathan:~$ 
```
So like usual, we `disassemble` the `main method` using `GDB`. It'll give us the `assembly code` which I can compile and put in `ghidra` and we get the following C code:

```c

undefined4 main(int param_1,undefined4 *param_2)

{
  int iVar1;
  __uid_t __euid;
  __uid_t __ruid;
  
  if (param_1 != 2) {
    printf((char *)0x804a008,*param_2);
                    /* WARNING: Subroutine does not return */
    exit(-1);
  }
  iVar1 = atoi((char *)param_2[1]);
  if (iVar1 == 0x1bd3) {
    __euid = geteuid();
    __ruid = geteuid();
    setreuid(__ruid,__euid);
    system((char *)0x804a022);
  }
  else {
    puts((char *)0x804a02a);
  }
  return 0;
}
```
We can ignore the first if, because it's just checking if there is no paramter and it says `usage: ./leviathan6 <4 digit code>`. In the second if, we can see that it checks if iVar1(which is the paramter we pass , the pin code) is equal to certain value, that value is `0x1bd3` in HEX and `7123` in DEC. 

```bash 
leviathan6@leviathan:~$ ./leviathan6 7123
$ whoami
leviathan7
$ 
```
From here we can just find the password in `/etc/leviathan_pass/leviathan7`.

**NOTE**: Usually when dealing with decompiled code, I would rename the variables to make the code more readable and easier to understand (e.g. iVar I would call it PIN or Param). But in here the code is very small so we can work with it raw.