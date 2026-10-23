---
title: "Level 3"
---

* Login : ssh leviathan3@leviathan.labs.overthewire.org -p 2223

Like the previous levels, in the home directory you'll find a `setuid ELF` called `level3` in this case. When you run it, you are prompted to enter a password :
```bash
leviathan3@leviathan:~$ ./level3 
Enter the password> password?
bzzzzzzzzap. WRONG
leviathan3@leviathan:~$ 
```

Unfortunately the passowrd wasn't password. So we keep digging. We can `disassemble` the binary and we get :
```bash 
   0x080492a9 <+0>:     lea    0x4(%esp),%ecx
   0x080492ad <+4>:     and    $0xfffffff0,%esp
   0x080492b0 <+7>:     push   -0x4(%ecx)
   0x080492b3 <+10>:    push   %ebp
   0x080492b4 <+11>:    mov    %esp,%ebp
   0x080492b6 <+13>:    push   %ecx
   0x080492b7 <+14>:    sub    $0x44,%esp
   0x080492ba <+17>:    mov    %gs:0x14,%eax
   0x080492c0 <+23>:    mov    %eax,-0xc(%ebp)
   0x080492c3 <+26>:    xor    %eax,%eax
   0x080492c5 <+28>:    movl   $0x626d6f62,-0x35(%ebp)
   0x080492cc <+35>:    movl   $0x646162,-0x32(%ebp)
   0x080492d3 <+42>:    movl   $0x732e2e2e,-0x20(%ebp)
   0x080492da <+49>:    movl   $0x33726333,-0x1c(%ebp)
   0x080492e1 <+56>:    movw   $0x74,-0x18(%ebp)
   0x080492e7 <+62>:    movl   $0x6f6e3068,-0x2e(%ebp)
   0x080492ee <+69>:    movl   $0x33336f,-0x2b(%ebp)
   0x080492f5 <+76>:    movl   $0x616b616b,-0x27(%ebp)
   0x080492fc <+83>:    movl   $0x616b61,-0x24(%ebp)
   0x08049303 <+90>:    movl   $0x2e32332a,-0x16(%ebp)
   0x0804930a <+97>:    movl   $0x785b2a32,-0x12(%ebp)
   0x08049311 <+104>:   movw   $0x5d,-0xe(%ebp)
   0x08049317 <+110>:   sub    $0x8,%esp
   0x0804931a <+113>:   lea    -0x27(%ebp),%eax
   0x0804931d <+116>:   push   %eax
   0x0804931e <+117>:   lea    -0x2e(%ebp),%eax
   0x08049321 <+120>:   push   %eax
   0x08049322 <+121>:   call   0x8049030 <strcmp@plt>
   0x08049327 <+126>:   add    $0x10,%esp
   0x0804932a <+129>:   test   %eax,%eax
   0x0804932c <+131>:   jne    0x8049335 <main+140>
   0x0804932e <+133>:   movl   $0x1,-0x3c(%ebp)
   0x08049335 <+140>:   sub    $0xc,%esp
   0x08049338 <+143>:   push   $0x804a037
   0x0804933d <+148>:   call   0x8049050 <printf@plt>
   0x08049342 <+153>:   add    $0x10,%esp
   0x08049345 <+156>:   call   0x80491d6 <do_stuff>
   0x0804934a <+161>:   mov    $0x0,%eax
   0x0804934f <+166>:   mov    -0xc(%ebp),%edx
   0x08049352 <+169>:   sub    %gs:0x14,%edx
   0x08049359 <+176>:   je     0x8049360 <main+183>
   0x0804935b <+178>:   call   0x8049070 <__stack_chk_fail@plt>
   0x08049360 <+183>:   mov    -0x4(%ebp),%ecx
   0x08049363 <+186>:   leave
   0x08049364 <+187>:   lea    -0x4(%ecx),%esp
   0x08049367 <+190>:   ret
```

You can try to read the assembly code, OR, we can put it in a `decompiler` and we get 
```c
#include <stdio.h>
#include <string.h>
 
void do_stuff(void);
 
int main(void)
{
    /* Stack buffers decoded from the immediate stores in the assembly.
       Little-endian byte order is preserved in the decoded strings. */
    char s_bombad[]   = "bombad";      /* -0x35(%ebp) */
    char s_h0no33[]   = "h0no33";      /* -0x2e(%ebp) */
    char s_kakaka[]   = "kakaka";      /* -0x27(%ebp) */
    char s_dots[]     = "...s3cr3t";   /* -0x20(%ebp) */
    char s_pattern[]  = "*32.2*[x]";   /* -0x16(%ebp) */
 
    int flag = 0;                      /* -0x3c(%ebp) */
 
    (void)s_bombad;
    (void)s_dots;
    (void)s_pattern;
 
    /* strcmp(-0x27(%ebp), -0x2e(%ebp)) is called with the second argument
       pushed first, so the comparison is strcmp(s_h0no33, s_kakaka). */
    if (strcmp(s_h0no33, s_kakaka) == 0) {
        flag = 1;
    }
 
    /* Original string at 0x804a037 is not present in the provided listing;
       a placeholder is used here. */
    printf("%s\n", "<string at 0x804a037>");
 
    do_stuff();
 
    (void)flag;
    return 0;
}
``` 

We can see a few important things here but we can dig deeper and view the `do_stuff` method:
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
 
void do_stuff(void)
{
    /* Expected passphrase: bytes 0x706c6e73, 0x746e6972, 0x000a6674 */
    char passwd[11] = "snlprintf\n";
    char buf[256];
 
    if (fgets(buf, sizeof(buf), stdin) != NULL) {
        /* Original code ignores fgets failure; the comparison still runs */
    }
 
    if (strcmp(buf, passwd) == 0) {
        /* String literal at 0x804a008 (not present in the disassembly) */
        puts("<success message>");
 
        uid_t ruid = geteuid();
        uid_t euid = geteuid();
        setreuid(ruid, euid);
 
        /* Command string literal at 0x804a01c (not present in the disassembly) */
        system("<command>");
    } else {
        /* String literal at 0x804a024 (not present in the disassembly) */
        puts("<failure message>");
    }
}
```

When viewing this code, I assume everyone would see the `passwd[]` variable and assume that's the password and go try it and hope for the best, and that's what I did. And it was correct. Yay, no need to read the code and try to understand. `EFFICIENCY`.

```bash
leviathan3@leviathan:~$ ./level3 
Enter the password> snlprintf
[You've got shell]!
$ whoami
leviathan4
$ 
```
And from this point we can just `cat` the password in `/etc/leviathan_pass/leviathan4`
