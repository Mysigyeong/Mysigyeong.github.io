---
sort: 2
---

# collision

home 디렉토리에 col.c파일을 확인하면 소스코드를 볼 수 있다.

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}
```

20byte읽어서 4byte씩 쪼갠 후에 int형 5개로 해석해서 모두 더한 값이 0x21DD09EC랑 같으면 된다.

```bash
col@pwnable:~$ ./col `python -c "print('\xc9\xce\xc5\x06'*4+'\xc8\xce\xc5\x06')"`
```

```tip
## 알게 된 점

음..... 딱히...?
```