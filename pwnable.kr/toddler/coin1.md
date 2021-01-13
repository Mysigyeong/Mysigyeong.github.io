---
sort: 11
---

# coin1

nc pwnable.kr 9007을 통해 접속하면 아래와 같이 뜬다.

```
    ---------------------------------------------------
    -              Shall we play a game?              -
    ---------------------------------------------------

    You have given some gold coins in your hand
    however, there is one counterfeit coin among them
    counterfeit coin looks exactly same as real coin
    however, its weight is different from real one
    real coin weighs 10, counterfeit coin weighes 9
    help me to find the counterfeit coin with a scale
    if you find 100 counterfeit coins, you will get reward :)
    FYI, you have 60 seconds.

    - How to play -
    1. you get a number of coins (N) and number of chances (C)
    2. then you specify a set of index numbers of coins to be weighed
    3. you get the weight information
    4. 2~3 repeats C time, then you give the answer

    - Example -
    [Server] N=4 C=2     # find counterfeit among 4 coins with 2 trial
    [Client] 0 1         # weigh first and second coin
    [Server] 20            # scale result : 20
    [Client] 3            # weigh fourth coin
    [Server] 10            # scale result : 10
    [Client] 2             # counterfeit coin is third!
    [Server] Correct!

    - Ready? starting in 3 sec... -
```

가짜동전 하나 찾는 유명한 문제다.<br>
Binary Search를 이용하면 쉽게 풀 수 있을 것 같다.<br>
동전을 반 뚝 잘라서 전체 무게가 10의 배수인지 확인하고 10의 배수가 아닌 부분을 가지고 이어서 반 뚝 잘라서 또 재는 것을 반복한다.<br>
여기서 중요한점은 저울질을 C번 모두 한 후에 그 다음에 답을 제출하는 방식이라는 것이다. 저울질하는 중간에 답이 나왔다고 해서 바로 다음으로 넘어가지 않는다.<br>
pwntools를 이용해서 코드를 짜면 되겠다.<br><br>
원격접속하면 느리기 때문에 pwnable.kr에 접속한 후 /tmp폴더 내부에 스크립트를 짜서 넣고 localhost로 돌렸다.

```py
from pwn import *
import time
import re

p = remote("localhost", 9007)
print(p.recvuntil("- Ready? starting in 3 sec... -"))
print(p.recvline())
print(p.recvline())

for _ in range(100):
    problem = p.recvline()

    numbers = re.findall("\d+", problem)

    n = int(numbers[0])
    c = int(numbers[1])
    start = 0
    end = n
    index_list = [i for i in range(n)]

    for _ in range(c):
        if start + 1 == end:
            mid = end
        else:
            mid = (start + end) // 2
        test_index_list = index_list[start:mid]
        message = ' '.join(map(str, test_index_list))
        p.sendline(message)

        response = p.recvline()

        response = int(re.findall('\d+', response)[0])

        if start + 1 == end:
            continue

        if response % 10 != 0:
            end = mid
        else:
            start = mid

    p.sendline(str(index_list[start]))
    p.recvline()

p.interactive()
```

```tip
## 알게 된 점

binary search
```