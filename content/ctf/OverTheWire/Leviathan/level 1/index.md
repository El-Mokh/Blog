---
title: "Level 1"
---

* Login : ssh leviathan1@leviathan.labs.overthewire.org -p 2223

After connecting to the machine, you'll find and `executable ELF` called `check`. When you execute it, it prompts you for a password. So a pretty educated guess would be that it waiting for a specific `password` to spit out the "flag" or at least move on to the next part of the puzzle.

My plan of aftion was using `gdb` and `disassemble` the main function.
I'll be presented with assembly. I usually just put it in a C translator or ghidra or something, and sometimes decode the assembly itself as practice. Anyway, here is both the assembly and C code : 

```text 
 0x080491d6 <+0>:     lea    0x4(%esp),%ecx
   0x080491da <+4>:     and    $0xfffffff0,%esp
   0x080491dd <+7>:     push   -0x4(%ecx)
   0x080491e0 <+10>:    push   %ebp
   0x080491e1 <+11>:    mov    %esp,%ebp
   0x080491e3 <+13>:    push   %ebx
   0x080491e4 <+14>:    push   %ecx
   0x080491e5 <+15>:    sub    $0x20,%esp
   0x080491e8 <+18>:    mov    %gs:0x14,%eax
   0x080491ee <+24>:    mov    %eax,-0xc(%ebp)
   0x080491f1 <+27>:    xor    %eax,%eax
   0x080491f3 <+29>:    movl   $0x786573,-0x20(%ebp)
   0x080491fa <+36>:    movl   $0x72636573,-0x13(%ebp)
   0x08049201 <+43>:    movl   $0x746572,-0x10(%ebp)
   0x08049208 <+50>:    movl   $0x646f67,-0x1c(%ebp)
   0x0804920f <+57>:    movl   $0x65766f6c,-0x18(%ebp)
   0x08049216 <+64>:    movb   $0x0,-0x14(%ebp)
   0x0804921a <+68>:    sub    $0xc,%esp
   0x0804921d <+71>:    push   $0x804a008
   0x08049222 <+76>:    call   0x8049050 <printf@plt>
   0x08049227 <+81>:    add    $0x10,%esp
   0x0804922a <+84>:    call   0x8049060 <getchar@plt>
   0x0804922f <+89>:    mov    %al,-0x24(%ebp)
   0x08049232 <+92>:    call   0x8049060 <getchar@plt>
   0x08049237 <+97>:    mov    %al,-0x23(%ebp)
   0x0804923a <+100>:   call   0x8049060 <getchar@plt>
   0x0804923f <+105>:   mov    %al,-0x22(%ebp)
   0x08049242 <+108>:   movb   $0x0,-0x21(%ebp)
   0x08049246 <+112>:   sub    $0x8,%esp
   0x08049249 <+115>:   lea    -0x20(%ebp),%eax
   0x0804924c <+118>:   push   %eax
   0x0804924d <+119>:   lea    -0x24(%ebp),%eax
   0x08049250 <+122>:   push   %eax
   0x08049251 <+123>:   call   0x8049030 <strcmp@plt>
   0x08049256 <+128>:   add    $0x10,%esp
   0x08049259 <+131>:   test   %eax,%eax
   0x0804925b <+133>:   jne    0x8049288 <main+178>
   0x0804925d <+135>:   call   0x8049080 <geteuid@plt>
   0x08049262 <+140>:   mov    %eax,%ebx
   0x08049264 <+142>:   call   0x8049080 <geteuid@plt>
   0x08049269 <+147>:   sub    $0x8,%esp
   0x0804926c <+150>:   push   %ebx
   0x0804926d <+151>:   push   %eax
   0x0804926e <+152>:   call   0x80490b0 <setreuid@plt>
   0x08049273 <+157>:   add    $0x10,%esp
   0x08049276 <+160>:   sub    $0xc,%esp
   0x08049279 <+163>:   push   $0x804a013
   0x0804927e <+168>:   call   0x80490a0 <system@plt>
   0x08049283 <+173>:   add    $0x10,%esp
   0x08049286 <+176>:   jmp    0x8049298 <main+194>
   0x08049288 <+178>:   sub    $0xc,%esp
   0x0804928b <+181>:   push   $0x804a01b
   0x08049290 <+186>:   call   0x8049090 <puts@plt>
   0x08049295 <+191>:   add    $0x10,%esp
   0x08049298 <+194>:   mov    $0x0,%eax
   0x0804929d <+199>:   mov    -0xc(%ebp),%edx
   0x080492a0 <+202>:   sub    %gs:0x14,%edx
   0x080492a7 <+209>:   je     0x80492ae <main+216>
   0x080492a9 <+211>:   call   0x8049070 <__stack_chk_fail@plt>
   0x080492ae <+216>:   lea    -0x8(%ebp),%esp
   0x080492b1 <+219>:   pop    %ecx
   0x080492b2 <+220>:   pop    %ebx
   0x080492b3 <+221>:   pop    %ebp
   0x080492b4 <+222>:   lea    -0x4(%ecx),%esp
   0x080492b7 <+225>:   ret
```

```text 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void) {
    char word1[] = "sex";
    char word2[] = "god";
    char word3[] = "love";
    char word4[] = "secret";
    (void)word2;
    (void)word3;
    (void)word4;

    char input[4];

    /* Original prompt string at 0x804a008 (text not recoverable from the disassembly) */
    printf("Enter input: ");

    input[0] = (char)getchar();
    input[1] = (char)getchar();
    input[2] = (char)getchar();
    input[3] = '\0';

    if (strcmp(input, word1) == 0) {
        uid_t uid = geteuid();
        setreuid(uid, uid);

        /* Command string at 0x804a013 (assumed to be a shell command, e.g. "/bin/sh") */
        system("/bin/sh");
    } else {
        /* Message string at 0x804a01b (text not recoverable from the disassembly) */
        puts("Access denied.");
    }

    return 0;
}
```


As you can see, the password is `sex`. And the C code actually reveals something else. When you enter the right password, you'll probably be dopped in /bin/sh. And indeed that was true. And we can just cat the password for the next user from /etc/leviathan_pass