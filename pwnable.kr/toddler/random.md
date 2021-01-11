---
sort: 6
---

# random

home 디렉토리에 random.c 파일을 통해 소스코드를 볼 수 있다.

```c
#include <stdio.h>

int main(){
        unsigned int random;
        random = rand();        // random value!

        unsigned int key=0;
        scanf("%d", &key);

        if( (key ^ random) == 0xdeadbeef ){
                printf("Good!\n");
                system("/bin/cat flag");
                return 0;
        }

        printf("Wrong, maybe you should try 2^32 cases.\n");
        return 0;
}
```

랜덤 seed가 변경되지 않으니 random에 똑같은 값만 들어갈 것이다.

```gdb
(gdb) disas main
Dump of assembler code for function main:
   0x00000000004005f4 <+0>:     push   %rbp
   0x00000000004005f5 <+1>:     mov    %rsp,%rbp
   0x00000000004005f8 <+4>:     sub    $0x10,%rsp
   0x00000000004005fc <+8>:     mov    $0x0,%eax
   0x0000000000400601 <+13>:    callq  0x400500 <rand@plt>
   0x0000000000400606 <+18>:    mov    %eax,-0x4(%rbp)
   
   ... (후략) ...

(gdb) b* 0x0000000000400606
Breakpoint 1 at 0x400606

(gdb) r
Starting program: /home/random/random

Breakpoint 1, 0x0000000000400606 in main ()
(gdb) i r rax
rax            0x6b8b4567       1804289383
```

random값이 0x6b8b4567이기 때문에 key에는 3039230856(0x6b8b4567 ^ 0xdeadbeef)이 들어가야 한다.<br>
scanf에서 10진수 format으로 받기 때문에 10진수로 넣어야한다.

```bash
random@pwnable:~$ ./random
3039230856
```

```tip
## 알게 된 점

random값 쓸 땐 seed를 바꿔주자.
```