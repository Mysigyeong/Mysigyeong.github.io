---
sort: 2
---

# Level 2

```note
id : level2

password : hacker or cracker
```

```bash
[level2@ftz level2]$ cat hint

텍스트 파일 편집 중 쉘의 명령을 실행시킬 수 있다는데...
```

level1과 마찬가지로 setuid가 걸려있는 파일을 찾자.

```bash
[level2@ftz level2]$ find / -user level3 -perm +4000 2> /dev/null
/usr/bin/editor
[level2@ftz level2]$ ls -al /usr/bin/editor
-rwsr-x---    1 level3   level2      11651 Sep 10  2011 /usr/bin/editor
```

setuid가 걸려있기 때문에 editor를 실행하면 level3의 권한을 얻게된다.<br>
editor를 실행하면 VIM이 실행이 된다. VIM은 Linux에서 사용되는 메모장이다.<br>
a, i 등의 키를 이용하여 텍스트 수정 모드로 들어갈 수 있고 esc키를 이용하여 명령어 모드로 들어갈 수 있다.<br>
명령어모드에서 :!를 이용하면 shell 명령어를 사용할 수 있다.

```bash
:!my-pass
```

```tip
## 알게 된 점

VIM 사용법
```