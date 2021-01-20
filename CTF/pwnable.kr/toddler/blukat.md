---
sort: 20
---

# blukat

home 디렉토리에 blukat.c파일에 소스코드가 들어있다.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
char flag[100];
char password[100];
char* key = "3\rG[S/%\x1c\x1d#0?\rIS\x0f\x1c\x1d\x18;,4\x1b\x00\x1bp;5\x0b\x1b\x08\x45+";
void calc_flag(char* s){
        int i;
        for(i=0; i<strlen(s); i++){
                flag[i] = s[i] ^ key[i];
        }
        printf("%s\n", flag);
}
int main(){
        FILE* fp = fopen("/home/blukat/password", "r");
        fgets(password, 100, fp);
        char buf[100];
        printf("guess the password!\n");
        fgets(buf, 128, stdin);
        if(!strcmp(password, buf)){
                printf("congrats! here is your flag: ");
                calc_flag(password);
        }
        else{
                printf("wrong guess!\n");
                exit(0);
        }
        return 0;
}
```

password파일을 읽어오는 것을 확인할 수 있다. 그래서 혹시나해서 password파일을 열어보았다.<br>
근데 열렸다.<br><br>

### ?

```bash
blukat@pwnable:~$ vi password
cat: password: Permission denied
```

내용물은 cat으로 열었을 때 권한때문에 안 열린 것처럼 하려고 훼이크를 쳐놨다.<br>
근데 나는 vi로 열어버렸다.<br>
왜 열리나 확인해보니 blukat 계정에 blukat_pwn이 그룹권한으로 있었다.

```bash
blukat@pwnable:~$ ls -al password
-rw-r----- 1 root blukat_pwn 33 Jan  6  2017 password
blukat@pwnable:~$ id
uid=1104(blukat) gid=1104(blukat) groups=1104(blukat),1105(blukat_pwn)
```

따라서 그냥 password를 redirect시켰다.

```bash
blukat@pwnable:~$ ./blukat < password
```

```tip
## 알게 된 점

group id도 잘 보자.
```