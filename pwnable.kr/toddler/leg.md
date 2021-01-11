---
sort: 8
---

# leg

leg.c 파일을 통해 소스코드를 볼 수 있다.<br>
leg.asm 파일을 통해 어셈블리어를 볼 수 있다.

```c
#include <stdio.h>
#include <fcntl.h>
int key1(){
	asm("mov r3, pc\n");
}
int key2(){
	asm(
	"push	{r6}\n"
	"add	r6, pc, $1\n"
	"bx	r6\n"
	".code   16\n"
	"mov	r3, pc\n"
	"add	r3, $0x4\n"
	"push	{r3}\n"
	"pop	{pc}\n"
	".code	32\n"
	"pop	{r6}\n"
	);
}
int key3(){
	asm("mov r3, lr\n");
}
int main(){
	int key=0;
	printf("Daddy has very strong arm! : ");
	scanf("%d", &key);
	if( (key1()+key2()+key3()) == key ){
		printf("Congratz!\n");
		int fd = open("flag", O_RDONLY);
		char buf[100];
		int r = read(fd, buf, 100);
		write(0, buf, r);
	}
	else{
		printf("I have strong leg :P\n");
	}
	return 0;
}
```

key1, key2, key3에서 뱉는 값들만 찾아내면 끝나는 것 같다.

```gdb
(gdb) disass key1
Dump of assembler code for function key1:
   0x00008cd4 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cd8 <+4>:	add	r11, sp, #0
   0x00008cdc <+8>:	mov	r3, pc
   0x00008ce0 <+12>:	mov	r0, r3
   0x00008ce4 <+16>:	sub	sp, r11, #0
   0x00008ce8 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008cec <+24>:	bx	lr
End of assembler dump.
```

key1에서는 pc값을 반환한다. pc는 x86의 rip와 비슷한 녀석인데, arm에서는 pipe 깊이가 2level이라고 했던가.. 아무튼 그래서 pc값은 해당 코드가 실행될 때 다음 instruction이 아닌 그 다음 instruction의 주소를 가리킨다고 한다.<br>
따라서 key1은 0x00008ce4이다.

```gdb
(gdb) disass key2
Dump of assembler code for function key2:
   0x00008cf0 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cf4 <+4>:	add	r11, sp, #0
   0x00008cf8 <+8>:	push	{r6}		; (str r6, [sp, #-4]!)
   0x00008cfc <+12>:	add	r6, pc, #1
   0x00008d00 <+16>:	bx	r6
   0x00008d04 <+20>:	mov	r3, pc
   0x00008d06 <+22>:	adds	r3, #4
   0x00008d08 <+24>:	push	{r3}
   0x00008d0a <+26>:	pop	{pc}
   0x00008d0c <+28>:	pop	{r6}		; (ldr r6, [sp], #4)
   0x00008d10 <+32>:	mov	r0, r3
   0x00008d14 <+36>:	sub	sp, r11, #0
   0x00008d18 <+40>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d1c <+44>:	bx	lr
End of assembler dump.
```

r3에 pc값(0x00008d08)을 대입, 그 후 4를 더함. 따라서 key2는 0x00008d0c다.<br>
참고로 중간에 pc값에 1을 더하는 것은 thumb모드로 변환하기 위해서 lsb를 set하는 것이다.

```gdb
(gdb) disass key3
Dump of assembler code for function key3:
   0x00008d20 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008d24 <+4>:	add	r11, sp, #0
   0x00008d28 <+8>:	mov	r3, lr
   0x00008d2c <+12>:	mov	r0, r3
   0x00008d30 <+16>:	sub	sp, r11, #0
   0x00008d34 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d38 <+24>:	bx	lr
End of assembler dump.

(gdb) disass main
Dump of assembler code for function main:

   ... (중략) ...

   0x00008d68 <+44>:	bl	0x8cd4 <key1>
   0x00008d6c <+48>:	mov	r4, r0
   0x00008d70 <+52>:	bl	0x8cf0 <key2>
   0x00008d74 <+56>:	mov	r3, r0
   0x00008d78 <+60>:	add	r4, r4, r3
   0x00008d7c <+64>:	bl	0x8d20 <key3>
   0x00008d80 <+68>:	mov	r3, r0

   ... (중략) ...
   
End of assembler dump.
```

key3은 lr이다. x86은 return address를 메모리 내부에 보관하지만, arm에서는 lr 레지스터에 보관한다.<br>
따라서 key3은 key3함수의 return address인 0x00008d80이다.<br><br>
따라서 key는 key1 + key2 + key3 = 0x00008ce4 + 0x00008d0c + 0x00008d80 = 108400이다.<br>
scanf에서 10진수로 입력을 요구하기 때문에 10진수로 써야한다.

```bash
/ $ ./leg
Daddy has very strong arm! : 108400
```

```tip
## 알게 된 점

arm
```