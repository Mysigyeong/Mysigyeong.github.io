---
sort: 2
---

# The Lord Of The Bufferoverflow

해커스쿨 lob 풀이.

```tip
버전이 너무 낮아서 bash에서 0xff를 입력하면 0x00으로 들어가는 버그가 존재. 따라서 root로 로그인하여 기본 쉘을 bash에서 bash2로 변경해야함.

root 비밀번호 : hackerschoolbof

/etc/passwd에 각 사용자의 default shell을 설정할 수 있으니 bash2로 바꾸자.

그 후 ifconfig로 ip확인한 후, telnet으로 접속하자.
```

{% include list.liquid all=true %}
