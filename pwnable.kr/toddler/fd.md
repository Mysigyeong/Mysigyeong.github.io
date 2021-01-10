---
sort: 1
---

# fd

home 디렉토리에 fd.c파일을 확인하면 소스코드를 볼 수 있다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
                printf("good job :)\n");
                system("/bin/cat flag");
                exit(0);
        }
        printf("learn about Linux file IO\n");
        return 0;

}
```

fd는 file descriptor의 약자로 fd가 0일땐 stdin, fd가 1일땐 stdout, fd가 2일땐 stderr를 나타낸다.
따라서 fd값이 0이 되도록 argv[1]에 4660(0x1234, atoi는 10진수로만 해석함)을 넣고 read를 통해 LETMEWIN을 입력해주면 된다.

```bash
fd@pwnable:~$ ./fd 4660
LETMEWIN
```

```tip
## 알게 된 점

fd
```