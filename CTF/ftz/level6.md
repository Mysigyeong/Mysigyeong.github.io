---
sort: 6
---

# Level 6

```note
id : level6

password : what the hell
```

```bash
hint - 인포샵 bbs의 텔넷 접속 메뉴에서 많이 사용되던 해킹 방법이다.
```

level6에 접속하자마자 위와 같이 hint가 나온다.<br>
엔터를 누르면 아래와 같이 텔넷 접속 서비스가 나온다.

```bash
  #####################################
  ##                                 ##
  ##         텔넷 접속 서비스        ##
  ##                                 ##
  ##                                 ##
  ##     1. 하이텔     2. 나우누리   ##
  ##     3. 천리안                   ##
  ##                                 ##
  #####################################

접속하고 싶은 bbs를 선택하세요 : 
```

하이텔 나우누리 천리안은 모두 과거의 산물이어서 지금은 접속도 안된다.<br>
그리고 우리는 저 사이트들로 접속하는 것이 목표가 아니라 shell을 획득하는 것이 목표다.<br>
텔넷 접속 서비스 프로그램을 종료하면 shell을 획득할 수 있을 것으로 보인다.<br>
따라서 `Ctrl + c`를 통해 KeyboardInterrupt 시그널을 날렸다. 그러나 텔넷 접속 서비스화면에서는 `Can't use ctrl+c`라는 문구가 출력되면서 interrupt에 대한 핸들링이 되어있었다. 따라서 `Ctrl + \`을 통해서 다른 시그널을 통해서 종료하는 것을 시도했다. 하지만 shell과의 접속 자체가 끊겨버렸다.<br>
텔넷 접속 서비스로 넘어가기 전, hint에서 `Ctrl + c`를 해야했다.<br>
hint단계에서 interrupt를 날려서 shell을 획득할 수 있었고, level6의 home directory에 password파일이 존재했다.

## 추가

.bashrc는 bash가 처음에 실행이 될 때 실행할 명령어들을 담아놓은 파일이다. 아래의 .bashrc를 살펴보면 `./tn`이 자동으로 실행되도록 한 것을 볼 수 있다.

```
# .bashrc

# User specific aliases and functions

# Source global definitions
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi
export PS1="[\u@\h \W]\$ "
./tn
logout
```

```tip
## 알게 된 점

시그널

.bashrc
```