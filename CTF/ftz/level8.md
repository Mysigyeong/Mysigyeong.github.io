---
sort: 8
---

# Level 8

```note
id : level8

password : break the world
```

```bash
[level8@ftz level8]$ cat hint

level9의 shadow 파일이 서버 어딘가에 숨어있다.
그 파일에 대해 알려진 것은 용량이 "2700"이라는 것 뿐이다.
```

level1에서 사용했던 find 명령어를 이용해 파일의 크기를 옵션으로 두고 찾으면 된다.<br> 
누가봐도 found.txt가 의심스럽다.

```bash
[level8@ftz level8]$ find / -size 2700c 2> /dev/null
/var/www/manual/ssl/ssl_intro_fig2.gif
/etc/rc.d/found.txt
/usr/share/man/man3/IO::Pipe.3pm.gz
/usr/share/man/man3/URI::data.3pm.gz
```

shadow 파일은 사용자의 비밀번호를 암호화해서 저장해놓은 것으로 다음과 같은 형식을 가진다.

```note
### shadow 파일 형식

1:2:3:4:5:6:7:8:9

1. 사용자 id
2. 비밀번호 정보
    $1$2$3
    1. hash 알고리즘
        * 1: md5
        * 2: BlowFish
        * 5: SHA256
        * 6: SHA512
    2. salt
    3. 비밀번호 hash값
3. 마지막 패스워드 변경날짜
4. 패스워드 최소 사용기간
5. 패스워드 최대 사용기간
6. 경고일
7. 비활성날짜
8. 만료일
9. 사용안함
```

중요한건 비밀번호 정보다. md5를 이용해 비밀번호에 salt값(vkY6sSlG)을 붙여서 hash값을 도출해 낸 것이 vkY6sSlG$6RyUXtNMEVGsfY7Xf0wps.라고 한다.<br>
hash는 일방향함수이기 때문에 복호화를 할 수 없다. 복호화를 하는 방법은 일일이 전부다 hash함수를 돌려보면서 어떤 input값이 어떤 hash값을 도출해내는지 보는 수 밖에 없다.

```
level9:$1$vkY6sSlG$6RyUXtNMEVGsfY7Xf0wps.:11040:0:99999:7:-1:-1:134549524
```

John the Ripper라는 password cracker 프로그램이 있어서 이를 활용했다.<br>
level9의 shadow파일 내용을 복사하여 level9.txt 파일로 만든 후 해당 파일을 넣고 돌렸다.

```powershell
PS C:\john-1.9.0-jumbo-1-win64> .\run\john.exe .\level9.txt
```

```tip
## 알게 된 점

shadow 파일

hash
```