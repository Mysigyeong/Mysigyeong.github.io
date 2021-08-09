---
sort: 8
---

# dragon

http://pwnable.kr/bin/dragon에 바이너리 파일이 있다.
nc pwnable.kr 9004에서 프로그램이 실행되고 있다.

<img src="/picture/pwnable.kr/dragon_1.png" width="1000"/>

main은 그냥 PlayGame함수 실행이 끝이다.<br>
PlayGame에서는 사용자로부터 1, 2, 3중 하나를 받는다.
1이면 Priest, 2면 Knight를 직업으로 선택할 수 있다. 3이면 SecretLevel로 갈 수 있다.<br><br>

<img src="/picture/pwnable.kr/dragon_2.png" width="1000"/>

SecretLevel함수로 들어오면 input을 또 받는데, 저게 Nice_뭐시기 문자열이랑 같으면 shell을 실행시켜준다.<br>
근데 scanf로 받는 문자 개수가 최대 10글자여서 이 함수자체는 못써먹는다.<br><br>

<img src="/picture/pwnable.kr/dragon_3.png" width="1000"/>

1이나 2를 넣으면, FightDragon함수로 가는데, 여기서 player정보랑 monster정보 넣고 PriestAttack또는 KnightAttack에서 싸운다.<br>
근데 이거 정상적인 방법으로는 못때려죽인다. 게임을 그지같이 만들어놨다.<br><br>

<img src="/picture/pwnable.kr/dragon_4.png" width="1000"/>

여기서 monster랑 player랑 struct field살펴보면 monster의 hp가 1byte인 것을 확인할 수 있다. 그리고 monster는 매 턴마다 heal을 하는 것을 볼 수 있다.<br>
따라서 priest로 계속해서 수비하면서 monster의 hp가 overflow가 될 때까지 뻐기면 된다.<br><br>

<img src="/picture/pwnable.kr/dragon_5.png" width="1000"/>

그리고 게임이 끝나면 PriestAttack함수에서 monster는 free가 된다. 근데 승리하면 FightDragon에서 이 free한걸 또 쓴다. 따라서 여기서 UAF가 터진다.<br>
이름 넣는공간의 첫번째 4byte에 SecretLevel함수에 있는 shell호출하는 부분의 주소를 넣으면 shell을 얻을 수 있다.

```python
from pwn import *

p = remote("pwnable.kr", 9004)
p.send(b'1\n' * 4 + b'3\n3\n2\n' * 4 + p32(0x08048dbf) + b'\n')
p.interactive()
```


```tip
## 알게 된 점

자료형 잘 보자.
```