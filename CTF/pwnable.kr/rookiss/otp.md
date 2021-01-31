---
sort: 4
---

# otp

home 디렉토리에 otp.c파일에 소스코드가 들어있다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

int main(int argc, char* argv[]){
    char fname[128];
    unsigned long long otp[2];

    if(argc!=2){
        printf("usage : ./otp [passcode]\n");
        return 0;
    }

    int fd = open("/dev/urandom", O_RDONLY);
    if(fd==-1) exit(-1);

    if(read(fd, otp, 16)!=16) exit(-1);
    close(fd);

    sprintf(fname, "/tmp/%llu", otp[0]);
    FILE* fp = fopen(fname, "w");
    if(fp==NULL){ exit(-1); }
    fwrite(&otp[1], 8, 1, fp);
    fclose(fp);

    printf("OTP generated.\n");

    unsigned long long passcode=0;
    FILE* fp2 = fopen(fname, "r");
    if(fp2==NULL){ exit(-1); }
    fread(&passcode, 8, 1, fp2);
    fclose(fp2);

    if(strtoul(argv[1], 0, 16) == passcode){
        printf("Congratz!\n");
        system("/bin/cat flag");
    }
    else{
        printf("OTP mismatch\n");
    }

    unlink(fname);
    return 0;
}
```

이 문제 정말 골때린다. 일단 random으로 file이름을 생성하고, random값을 집어넣은 뒤, argv[1]과 file 내용물을 비교한다.<br>
문제는, argv[1]을 사용자가 입력하는 시점이 random이름을 가진 file 자체가 만들어지기 이전이다.<br>
urandom에서 뽑아올 값 두개를 먼저 알아내야한다. 근데 알아낼 수 있으면 누가 이걸 쓸까. 이미 다 갈아엎어졌을 것이다. 따라서 이건 아닌거 같다.<br>
저 임시파일을 중간에 낚아채려고하니, 파일의 이름도 알 수 없고, 설령 낚아챘다고 하더라도, 값 비교 루틴으로 들어가기 전에 argv[1]의 값을 바꿔야 하는 상황이 온다. 말이 안된다.<br>
따라서 이 문제의 의도는 파일의 내용물을 고정시킬 방법을 모색하라는 것으로 보인다.<br>
구글링을 하니, ulimit 명령어를 이용해 프로세스가 만드는 파일의 크기를 제한하여 파일에 내용물을 쓰지 못하게 하는 방법이 있었다.<br><br>
여기는 나도 잘 모르는 부분이니 예제코드와 함께 상황이 어떻게 되는지 아래에 기술한다.

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv)
{
    FILE* fp = fopen("./tmp_file", "w");
    printf("fp = %p\n", fp);
    if (fp == NULL) exit(1);

    size_t write_length = fwrite("hello world", 1, 11, fp);
    printf("write_length = %d\n", write_length);

    int ret = fclose(fp);

    printf("ret = %d\n", ret);

    return 0;
}
```

```bash
# 정상으로 처리되는 상황
$ ./ulimit_test
fp = 0x55cc89d612a0
write_length = 11
ret = 0

$ cat tmp_file
hello world


# ulimit -f 0을 통해 만들어지는 file의 크기를 제한한 상황
$ ./ulimit.sh
fp = 0x55cc1bf712a0
write_length = 11
./ulimit.sh: line 5:  1230 File size limit exceeded./ulimit_test
# SIGXFSZ로 인해 프로세스가 중간에 종료되는 모습이다.

$ file tmp_file
tmp_file: empty


# SIGXFSZ까지 handling한 상황
$ ./ulimit.sh
fp = 0x5575a76792a0
write_length = 11
# fclose에 실패하여 EOF가 반환되는 모습이다.
ret = -1

$ file tmp_file
tmp_file: empty
```

shell script를 이용해서 trap과 ulimit을 사용했다.

```bash
#!/bin/bash

trap "" XFSZ
ulimit -f 0

/home/otp/otp 0
```

```tip
## 알게 된 점

SIGXFSZ

ulimit

fclose의 반환값도 유심히 보자.
```