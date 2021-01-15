---
sort: 9
---

# mistake

home 디렉토리에 있는 mistake.c 파일을 통해 소스코드를 볼 수 있다.

```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
        int i;
        for(i=0; i<len; i++){
                s[i] ^= XORKEY;
        }
}

int main(int argc, char* argv[]){

        int fd;
        if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
                printf("can't open password %d\n", fd);
                return 0;
        }

        printf("do not bruteforce...\n");
        sleep(time(0)%20);

        char pw_buf[PW_LEN+1];
        int len;
        if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
                printf("read error\n");
                close(fd);
                return 0;
        }

        char pw_buf2[PW_LEN+1];
        printf("input password : ");
        scanf("%10s", pw_buf2);

        // xor your input
        xor(pw_buf2, 10);

        if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
                printf("Password OK\n");
                system("/bin/cat flag\n");
        }
        else{
                printf("Wrong Password\n");
        }

        close(fd);
        return 0;
}
```

`fd=open("/home/mistake/password",O_RDONLY,0400) < 0`에서 비교연산자가 대입연산자보다 우선순위가 높기 때문에 fd에는 거짓을 나타내는 0이 들어가게 된다.<br>
fd가 0이라는 소리는 read를 했을 시 stdin에서 읽어오겠다는 소리기 때문에 `len=read(fd,pw_buf,PW_LEN) > 0`에서 stdin을 읽어 pw_buf가 채워진다.<br>
pw_buf2에서도 stdin을 통해 값을 읽어오니 한번은 10글자 대충 치고 또 한번은 그것의 xor 1값을 입력해주면 되겠다.

```bash
mistake@pwnable:~$ ./mistake
do not bruteforce...
BBBBBBBBBB
input password : CCCCCCCCCC
```

```tip
## 알게 된 점

연산자 우선순위
```