---
title: "Level 4"
---

* Login : ssh leviathan4@leviathan.labs.overthewire.org -p 2223

Well, for this level, you can't find and ELF in the home directory :(. BUT, there is a hidden directory called `.trash`, and that directory DOES have an `ELF`. The elf is called `bin` and when you run it it prints out a ``binary sequence`. My first instict was to `translate it to ASCII` and, to no surprise, I got the password.

```bash
leviathan4@leviathan:~/.trash$ ls -l
total 16
-r-sr-x--- 1 leviathan5 leviathan4 14936 Sep 26 21:56 bin
leviathan4@leviathan:~/.trash$ ./bin 
01000010 01110101 01100010 00111001 01100111 01011010 00110011 01000010 01000111 01010101 00001010 
leviathan4@leviathan:~/.trash$ 
```