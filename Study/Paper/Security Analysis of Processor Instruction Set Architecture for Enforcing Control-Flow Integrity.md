---
sort: 1
---

# Security Analysis of Processor Instruction Set Architecture for Enforcing Control-Flow Integrity

## ABSTRACT
Intel에서 control flow check하는 CET기능을 CPU에 적용.<br>
control flow 가지고 장난치는 ROP, JOP와 같은 공격이 있음.<br>
얘네 어케 막을 수 있는지 살펴보고 성능 측정해봤음.<br><br>

## 1. INTRODUCTION

DEP, SMEP, SMAP와 같은 mitigation들이 ROP, JOP, COP와 같은 공격이 나오게 함.

* DEP: Data Execution Prevention<br>
	데이터 영역 실행하지 못하도록 영역 지정하는 것.<br>
* SMEP: Supervisor Mode Execution Protection Enable<br>
	ring 단계 높은 곳에서 실행하면 fault.<br>
* SMAP: Supervisor Mode Access Protection Enable<br>
	ring 단계 높은 곳에서 사용하면 fault.<br>

위 세개를 막기 위해서 CFI가 논의되는데, Intel에서 CET를 도입했다.<br><br>
Ret addr보호를 위해서 shadow stack, indirect branch보호를 위해서 indirect branch tracking으로 구성됨.<br>
### CET shadow stack
software단의 unintended write가 shadow stack을 대상으로 이루어지지 않도록 보호받고있음.<br>
call을 수행할 때 shadow stack에도 ret addr를 씀. ret을 수행할 때 stack과 shadow stack상의 ret addr값을 비교. 같으면 정상. 다르면 error.<br>
### CET indirect branch tracking
indirect branch의 도착점에 endbranch 명령어를 둬서 도착점이 endbranch가 아닐경우 exception 발생하도록 함.<br>
Control Protection(CP) exception을 새로 도입해 CF에 대한 issue 따로 저장.<br>

### Threat model
1.	Arbitrary read write를 발견할 수 있는 공격자
2.	주소공간의 complete layout을 알아낼 수 있음.
3.	mem read write를 원하는 대로 반복 가능
4.	code 흐름을 원하는 대로 조작 및 프로그램 흐름 모니터링 가능.
5.	computation server로 data 보내서 payload 실행 가능
6.	processor의 register상황을 바꾸기 위해 code reuse를 통한 control transfer 가능.

### 공격자에 대한 가정
1.	NX로 인해 새로운 code를 공격자가 주입할 수는 없고, code는 모두 read only다.

### CET의 design goal과 constraint
1.	새로운 architecture asset(새로운 hardware register, memory 등)에 대해서 반드시 code reuse 및 memory safety error로부터 보호해야한다.
2.	CPU-enforced privilege level(user/supervisor)에 적용할 수 있어야 한다.
3.	상용 프로그램이 사용하는 CPU 모드들(32bit/64bit, hypervisor, system management mode, enclave, 등등)에 대해서도 적용 가능해야함.
4.	mode가 변하는 시점 또는 context switch하는 타이밍에도 control flow에 대한 protection이 이루어져야 한다.
5.	performance, memory usage, code size 변화에 대한 overhead를 최소화해야한다.
6.	ISA의 특정 코드들을 끼워 넣는 프로그래밍 언어는 피한다.<br>
```
There are many examples of embedding web languages and programming languages

* Embedding of CSS in HTML, SVG and other XML languages
* Embedding of Javascript in HTML for client dynamics
* Embedding of a programming language in HTML at the server-side

ASP: HTML embedding of Visual Basic fragments
JSP: HTML embedding of Java fragments
PHP: HTML embedding of C-like fragments
BRL: HTML embedding of Scheme fragments
```
7.	return address protection, instruction alignment만 딱 할 수 있을 정도의 최소한의 권한만 부여해야한다.
8.	stack과 function call에 대한 ABI를 유지한다.
9.	tail call, co-routine과 같은 common software construct에 어떠한 제한도 줘선 안된다.

## 2. CET SHADOW STACK
shadow stack은 control transfer operation에 의해 사용되는 return address를 저장하기 위한 두번째 stack으로, 일반적인 data는 넣지 않는다.

### 2.1. SSP register encoding and type safety
현재 shadow stack의 top을 가리키는 shadow stack pointer(SSP)가 있음.<br>
SSP를 보호하기 위해서 일반적인 기존 opcode의 operand로 직접적으로 사용할 수 없게 하고, shadow stack management instruction으로만 관리할 수 있게 함. 이를 통해 가젯 재사용을 통한 공격이 쉽게 되지 않도록 함.<br>
CET instruction이나 privilege transition과 같은 implicit ISA flow에서 만들어지는 값들의 type safety를 위하여 상호보완적인 CET instruction또는 complimentary ISA flow에 대해서만 해당 값을 사용할 수 있게 했다.<br>

### 2.2. Shadow stack for each privilege and exception delivery
CET는 각각의 privilege level(ring 0 ~ 3)에서 모두 control flow enforcement를 제공한다. ring 0 ~ 2를 모두 동일한 privilege level로 여겨서 supervisor mode로 동작한다.<br>
현재 task에서 privilege level 0, 1, 2를 사용하는 stack을 저장하기 위해 Task-State Segment(TSS) data structure가 사용됨. 여기를 포인터로 가리킴.<br>
intel 64bit architecture에서 서로 다른 privilege의 stack을 관리하기 위해 interrupt stack table(IST), interrupt-gate-descriptor(IDT)를 사용한다. IST는 3bit로 최대 7개의 pointer를 제공하며, IST에 저장된 index를 통해 IDT의 해당하는 부분에 저장된 stack pointer를 사용할 수 있다. 이러한 automatic stack switching기법을 확장하여 SSP는 새로운 model specific registers(MSRs)를 이용해 privilege에 따른 shadow stack을 관리하도록 했다.<br>
IA32_INTERRUPT_SSP_TABLE_ADDR MSR을 이용해 privilege를 바꾸는 interrupt가 발생했을 때 그에 맞는 SSP가 선택되도록 했다. ring 0가 가장 권한이 높고, 1, 2는 낮아서 ring 0에서만 privileged instruction을 사용할 수 있고 1, 2에서는 사용 못한다. 그러나 0, 1, 2모두 memory 접근에 대한 권한은 똑같기 때문에 0~2를 동일한 privilege로 여긴다.<br>

### 2.3. CALL operation
같은 code segment 내에서 call하는 near call을 할 땐 return address를 stack에 저장하고 jump한다. CET가 활성화되면 return address를 shadow stack에도 저장하고 jump한다.<br>
다른 code segment로 jump하는 far call을 할 땐 Code Segment(CS) selector와 return address를 stack에 넣고 jump한다. CET가 활성화되면 CS와 Linear Instruction Pointer(LIP; CS descriptor의 base와 logical return address를 이용해 계산된 값)과 SSP값을 shadow stack에도 저장하고 jump한다.<br>
shadow stack에는 항상 8byte push만 일어난다.<br>
이전의 SSP값을 shadow stack에 넣는 이유는 far CALL에 사용되었던 값이 near return에 잘못 사용되는 것을 방지하기 위함이다. shadow stack이 저장되어있는 곳을 가리키는 주소들은 non executable이기 때문이다.<br>
(아마도 near return으로 잘못 사용되면 SSP값을 return address로써 사용하게 될 테니 그러면 뻑이 나서 못쓰게끔 한다 그런 것 같다.)<br>
far call에서는 return address를 그냥 쓰지 않고 LIP를 사용하는 이유는 far call을 할 땐 64, 32비트간의 변환, privilege의 변환 등 return address를 그냥 사용하기에는 부적절한 상황들이 발생하기 때문에 LIP를 사용한다.<br>

### 2.4. RET/IRET operation
RET instruction은 CALL로 갔었던거 다시 되돌아오는 instruction, IRET instruction은 interrupt나 exception을 handling하는 procedure에서 return할 때 사용.<br>
CET가 켜져있으면, near RET는 shadow stack와 data stack에 있는 return address를 비교하고 둘이 다르면 error code NEAR-RET와 함께 control protection exception을 낸다.<br>
far RET/IRET을 할 땐, CET가 활성화 되어있을 때 shadow stack에서 return-SSP, LIP, CS값을 가져온다. (user space로의 transition이 아닐 경우)<br>
CS, LIP가 data stack의 CS와 return address와 맞지 않을 경우, FAR-RET/IRET 에러코드와 함께exception을 발생시킨다. 맞을 경우에는 SSP가 return-SSP값으로 설정이 된다.<br>
user space로 되돌아오기 위해서 RET/IRET가 사용된다면, IA32_PL3_SSP MSR을 사용해서 SSP를 만들고 return address verification이 이루어지지 않는다. user mode program은 OS를 trust boundary에 넣는다고 가정을 했기 때문에 OS가 정해준 값을 굳이 verification하지 않는 것이다.<br>

### 2.5. Write protecting the shadow stack
공격자는 어떻게든 shadow stack이 저장되어 있는 영역을 수정하려고 할 것이다. 그러나 shadow stack영역은 CALL, RET, CET 관련 instruction(shadow_stack_access)들만 사용할 수 있다. 일반적인 mov instruction으로는 사용불가.

#### 2.5.1 x86 paging protections
기존의 page architecture를 확장시켜서 같은 linear address space에다가 shadow stack page를 만들 수 있게 함. non writeable하고 dirty한 page를 CPU는 shadow stack page로써 여기게 함.<br>
기존 software가 사용하지 않던 page flag 상황을 사용함으로써 새로운 flag bit를 도입하는 것을 회피할 수 있게 됨. + non writeable로 page를 할당해서 malicious write로부터 막을 수 있음.<br>
shadow_stack_access명령어는 무조건 shadow page내의 shadow stack만 접근 가능. 그렇지 않으면 segmentation fault발생. 이를 이용해 SSP pivoting을 통해 shadow stack이 아닌 다른곳을 접근하는 것을 막음. (SSP가 writable한 page를 가리키게 한다던지 등등.)<br>
privilege에 맞는 shadow stack을 사용해야 함. privilege가 맞지 않으면 segmentation fault발생.<br>
CET가 활성화되어있을 땐, paging write protection을 끌 수 없게 강제됨.<br>

#### 2.5.2 second level page table protection
Virtual Machine Manager의 Extended Page Table(EPT)을 이용해서 OS/supervisor shadow stack을 malicious write로부터 보호 가능. supervisor shadow stack영역을 guest에서 접근하기 위해서는 EPT가 mapping한 guest상의 supervisor shadow stack영역을 통해서만 접근 가능.<br>

### 2.6. shadow stack tokens
shadow stack을 보호하기 위해 shadow page를 non writeable로 만들었고, shadow stack이 아닌 곳을 접근하려고 하면 segmentation fault를 발생시켰다. shadow stack을 가리키는 pointer는 독점적으로, 여러 프로세스에서 사용하면 segmentation fault가 발생한다.<br>
독점적으로만 사용할 수 있도록 하기 위해서 shadow stack바닥부분에 shadow stack token을 저장한다. 저장된 shadow stack token의 최하위 2비트를 flag로 사용한다.<br>

### 2.7. Processor initiated stack switch
shadow stack은 여러 프로세스가 동시에 사용할 수 없어야 한다. context switch로 인해서 shadow stack이 잘못 사용되는 것을 막기 위해서 supervisor shadow stack의 바닥 8byte에 supervisor shadow stack token이 있다.<br>
shadow stack token은 Bit 63:3 – 8 byte align된 token 자기 자신의 linear address가, Bit 2:1 – reserved. 반드시 0이어야 한다. Bit 0 – Busy bit. 여기가 set 되어있으면 해당 shadow stack을 사용하는 active processor가 존재하는 것이다. 이를 이용해서 shadow stack이 여러 processor에 의해 사용되는 것을 막는다.<br>
더 높은 privilege level의 stack으로 이동할 때, process는 아래와 같이 행동한다.

1. supervisor shadow stack token을 IA32_PL/0/1/2_SSP로부터 로드함.
2. busy bit와 reserved bit를 check함.
3. token에 있는 address와 MSR에 있는 address가지고 검증한다. (둘이 같은지 확인 == shadow stack이 비어있는지 확인. SSP가 shadow stack의 바닥에 존재하는 shadow stack token을 가리키고있는 상황)
4. 다 정상적이면 busy bit를 set하고 SSP 교체한다.

Processor가 낮은 privilege level로 return할 때 shadow stack도 낮은 privilege의 것으로 바꿔야 함. 현재 사용하고있던 shadow stack은 far RET/IRET를 통해 free가 된다.

1. supervisor shadow stack token을 가져온다.
2. busy bit는 1이고 reserved bit는 0인지 확인한다.
3. token에 있는 address와 SSP와 맞는지 확인. (shadow stack이 비어있는지 확인)
4. 검증 완료 후 token의 busy bit를 clear.

만약에 3번에서 shadow stack이 비어있지 않다면, valid call frame이 현재 프로세스에 존재하는 것이기 때문에 busy bit를 clear하지 않는다.

### 2.8. Shadow stack management instructions
#### 2.8.1 stack unwinding
stack unwinding: 말 그대로 스택을 풀어나가는 것. try catch문과 같이 예외를 처리할 때 예외 처리하는 루틴이 현재 루틴보다 상위에 있을 때 stack unwinding 사용.<br>
INCSSP n: SSP를 n * operand size of shadow stack만큼 증가시킨다. n은 최대 255로 제한하여 shadow page밖을 벗어날 수 없도록 했다.<br>
RDSSP: SSP register 값을 General Purpose Register로 읽어오는 명령어. – setjmp와 같은 명령어를 사용할 때 필요한 경우가 있다고 한다.<br>

#### 2.8.2 software initiated stack switching
context switch라던지 thread switch라던지 task가 바뀌면 stack도 바뀐다. shadow stack도 바뀌어야 한다. 따라서 이를 도와주는 RSTORSSP와 SAVPREVSSP명령어를 제공한다.<br>
RSTORSSP를 이용해 새로운 shadow stack의 shadow stack restore token을 verify 후 새로운 shadow stack으로 switch한다. 그 후, SAVEPREVSSP를 이용해 이전 shadow stack의 restore point를 저장한다. restore point는 이전 shadow stack의 top에 shadow stack restore token의 형식으로 저장된다. 또는 OS가 직접 새로운 shadow stack을 set up할 때 restore point를 만들 수 있다.<br>
shadow stack은 동시에 여러 process가 사용할 수 없기 때문에 동일한 shadow stack restore token은 한 번만 사용 가능하다.<br>
shadow stack restore token은 64bit값이다. Bit 63:2 – 4byte aligned SSP for which this restore point was created. token에 저장된 값 + 8또는 +12를 했을 때 새로운 shadow stack의 SSP값이 나와야 한다. bit:1 – reserved. 반드시 0이어야 한다. bit:0 – mode bit. 만약 0이면 RSTORSSP를 32bit mode에서 사용 가능. 1이면 64bit mode에서 사용 가능.<br>
shadow stack restore token은 SAVEPREVSSP 명령어에 의해 만들어진다.<br>
RSTORSSP 명령어는 아래와 같은 작업을 통해 shadow stack restore token을 검증하고 SSP를 바꾼다.

1. memory operand로 제공받은 주소를 이용해 shadow stack restore token을 가져온다.
2. reserved bit가 0인지 확인한다.
3. bit 63:2에 저장되어있는 SSP값이 token 주소 + 8또는 + 12인지 확인.
4. 현재 machine의 mode와 mode bit가 맞는지 확인한다.
5. 2~4가 통과되면 shadow stack restore token을 previous SSP token으로 바꾼다.
6. SSP값을 token의 주소로 바꿔서 stack을 바꾼다. 이제는 shadow stack의 top에 previous SSP token이 있는 상태다.

previous SSP token은 64bit값이다. Bit 63:2 – 바꾸기 이전의 shadow stack의 SSP값. Bit 1 – previous SSP token임을 알리기 위해서 1로 설정한다. Bit 0 – mode bit. 0일땐 SAVEPREVSSP를 32bit mode로, 1일땐 64bit mode로 사용해야함.<br>
SAVEPREVSSP 명령어는 이전 shadow stack에 shadow stack restore token을 집어넣는다. 현재 shadow stack의 top에 있는 previous SSP token을 사용하여 이전 shadow stack에 저장한다.

1. SSP가 8byte align되어있는지 확인.
2. previous SSP token을 shadow stack에서 pop한다.
3. bit:1이 1인지 확인.
4. mode bit확인.
5. pop해온 previous SSP token을 이용해서 이전 shadow stack의 SSP - 8또는 SSP - 12부분에 shadow stack restore token 넣어놓는다.

#### 2.8.3 shadow stack fixup
shadow stack을 고칠 수 있도록 관련된 명령어 제공.<br>
WRUSS(Writes User Shadow Stacks): privileged instruction으로 OS만 사용 가능. User shadow stack을 대상으로만 사용 가능.<br>
WRSS: user가 호출하면 user shadow stack을, supervisor가 호출하면 supervisor shadow stack을 대상으로 사용.<br>

#### 2.8.4 Fast system call support
SYSCALL SYSENTER와 같은 명령어로 OS를 호출하면 IA32_PL3_SSP MSR에다가 SSP를 저장하고 SSP를 0으로 set한다. SYSRET, SYSEXIT과 같은 명령어로 다시 user mode로 돌아오면 IA32_PL3_SSP MSR에 저장되어있던 값을 SSP로 불러온다.<br>
SSP값이 0으로 되었기 때문에 supervisor shadow stack을 사용하기 위해서 SETSSBSY 명령어를 사용해서 IA32_PL0_SSP에 있는 값을 이용해 supervisor shadow stack을 사용하도록 한다. SETSSBSY명령어는 supervisor shadow stack token을 검증하고, 검증에 성공하면 busy bit를 set하고 해당 supervisor shadow stack을 사용한다. 이 shadow stack을 deactivate하기 위해선 CLRSSBSY를 이용해야한다.

## 3. INDIRECT BRANCH TRACKING
indirect branch로 인해서 이상한 곳으로 뛰는 것을 막기 위해서 indirect branch의 target address부분에 ENDBR32또는 ENDBR64를 넣어놓는다. 만약 indirect branch를 했는데 ENDBR 명령어를 만나지 못하면 CP exception을 낸다. ENDBR은 CET가 활성화되지 않았을 때에는 NOP으로써 동작하도록 설계해서 CET 활성화 안된 CPU에서도 program을 원활하게 돌릴 수 있다. 뭐 특별한 register를 요구하는 것도 아니다.<br>
indirect branch의 목적지를 추적하기 위해서 user mode, supervisor mode 각각에서 동작하는 state machine을 만들었다.<br>
state machine은 평소에는 idle이었다가 branch하는 명령어를 만나면 wait for endbranch state로 갔다가 endbranch 제대로 만나면 다시 idle로, 제대로 못만나면 fault state로 간다.<br>
relative call, relative jump, conditional jump는 코드에 박혀있고 조작이 불가능하기 때문에 state machine이 얘네들에겐 반응 안한다.

### 3.1. NO-TRACK prefixed jmp/call
switch문과 같이 compiler가 전적으로 관리해서 jump target이 모두 유효한 상황에서는 굳이 endbranch를 사용할 필요가 없다. 위와 같은 이유로 endbranch를 사용하지 않는 jmp에는 NO-TRACK 접두사를 붙일 수 있다. 또한 exec, execv와 같이 민감한 함수들은 목적지가 정해져 있어서 이와 같을 때에도 endbranch를 사용하지 않게 할 수 있다.

### 3.2. ENDBRANCH Opcode selection
ENDBR32: F3 0F 1E FB, ENDBR64: F3 0F 1E FA. 실제 ENDBR가 아닌데도 불구하고 ENDBR로써 해석이 될 수 있는 코드가 발생하지 않도록 위와 같이 정했다고 한다.

## 4. SPECULATION SAFE PROPERTIES OF CET
spectre와 같은 공격에 당하지 않도록 speculative execution side-channel attack의 mitigate를 넣어둠.
Constraining execution at targets of RET
CET shadow stack이 활성화 되어있을 땐 ret address부분에 대해서 speculative execution을 하지 않도록 한다.

#### Speculation constraint on missing ENDBRANCH
WAIT_FOR_ENDBRANCH state라면 다음 instruction이 ENDBRANCH가 아닐경우 speculative execution이 제한된다.<br>
평소에 실행할 때도 indirect branch부분을 speculative하는 상황이 온다면, state가 WAIT_FOR_ENDBRANCH가 되는 것 + ENDBRANCH가 존재하지 않는다고 가정을 하고 진행하기 때문에 indirect branch의 목적지에 대한 실행이 제한된다.

## 5. RESULTS AND DISCUSSION
### 5.1. Performance
shadow stack도입으로 인해 call과 ret에서 추가적인 일을 할 경우에 발생하는 IPC손실은 평균 1.65%이다. 0.08~2.71%정도 나옴.<br>
indirect branch에 대한 수정으로 인한 손실은 비교해보니까 뭐 거의 없다고 한다.<br>
```warning
## 논문에 대한 개인적인 비평.

Performance에 실험환경이 명시되어있지 않다.
context switch로 인해 추가적으로 발생하는 shadow stack관리 루틴에 대해서는 이루어지지 않은 것 같다. 실제로 CET를 지원하는 CPU를 만들어서 한 것 같진 않고, call이랑 ret일때만 shadow stack에 추가적으로 push를 하거나 pop후 값 비교를 하도록 에뮬레이션을 한 것 같다. 또한 indirect branch에 대한 검사를 하기 위해 state machine을 추가적으로 돌려야 하는데, state machine을 도입하면 성능손실이 추가적으로 발생할 것 같다. 논문에서 indirect branch에 대한 handling에 대한 성능손실이 거의 없다고 표현한 이유는 CPU도 CET를 지원하는 CPU가 아니고, 어차피 CET모드가 활성화 되어있지 않는 한, ENDBRANCH는 NOP으로 인식되기 때문에 성능차이가 없는 것은 어찌보면 당연해보인다. 또한 CET모드에서는 speculative execution도 제한하는데, 이를 통해 추가적인 성능 손실이 발생할 것 같은데 performance 문단에서는 이에 대한 언급이 없다.
```

### 5.2. security metrics
shadow stack의 도입으로 인해서 ROP가 매우 힘들어졌다. ret할때마다 ret address에 대한 검증이 들어가기 때문. 따라서 공격자는 shadow stack을 어떻게든 조작해보려고 할텐데, shadow stack은 기본적으로 unwritable page에 존재하고, SSP pivoting을 해보려고 해도 SSP변경이 쉽지 않고, 변경한다고 하더라도 유효하지 않은 곳을 참조하면 바로 segmentation fault를 내뿜는다.<br>
indirect branch에 대한 mitigation으로 인해 endbranch가 무조건 있어야 하기 때문에 가젯을 찾기가 매우 힘들어졌다. shadow stack때문에 indirect branch 이후로도 control flow 이어나가기가 어려울 것이다.<br>
CPU단에서 제공하는 이러한 방어기법 뿐만 아니라 software적으로도 방어기법을 추가한다면 훨씬 공격이 어려워질 것이다.<br>

#### Average indirect target reduction(AIR)
indirect control transfer로 접근 가능한 프로그램의 영역을 계산해서 CFI가 얼마나 좋은지 판단하는데 사용되는 metric이다. 높으면 CFI가 좋은것이다. SPEC 돌려보니까 99.9%대로 나온다고 한다. CFI가 좋은듯.<br>

#### Linux Kernel Gadget Analysis
ROPgadget을 이용해서 Linux kernel의 가젯 분석. 가젯 많이 나왔는데 CET로 거의 다 막힘.<br>
kernel에 exported function들 18412개 있던데, 얘네 이용해서 COP할 수도 있겠는데, 얘네 함수가 대부분 억수로 길어서 side effect때문에 chaining하기 억수로 힘들 듯.<br>

```tip
## 알게 된 점

intel에서 CFI를 위해 CET를 도입.
Return address 보호를 위해 shadow stack 사용.
Indirect branch 보호를 위해 endbranch 사용.
```