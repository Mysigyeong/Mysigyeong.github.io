---
sort: 7
---

# Level 7

```note
id : level7

password : come together
```

```bash
[level7@ftz level7]$ cat hint

/bin/level7 명령을 실행하면, 패스워드 입력을 요청한다.

1. 패스워드는 가까운곳에..
2. 상상력을 총동원하라.
3. 2진수를 10진수를 바꿀 수 있는가?
4. 계산기 설정을 공학용으로 바꾸어라.
```

level7을 실행시켜보니 비밀번호입력을 요구했고, 모르기때문에 막 눌러보니 /bin/wrong.txt파일이 없다고 떴다.<br>
wrong.txt파일에 힌트가 있어야 할 것으로 여겨지는데 파일이 존재하지 않는다.

```bash
[level7@ftz level7]$ level7
Insert The Password : asdf
cat: /bin/wrong.txt: No such file or directory
```

원래는 아래와 같은 힌트가 주어져야한다고 한다.

```
--_--_-     --____-     ---_-__     --__-_-
```

각 자리가 7글자인 것을 보니 대놓고 ascii라고 외쳐대는 것 같다. -를 1, _을 0으로 놓고 이진수로써 해석을 하고, ascii코드에 대입하면 mate가 나온다.

```tip
## 알게 된 점

이진수

ascii
```