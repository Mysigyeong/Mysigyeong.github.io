---
sort: 2
---

# Meltdown: Reading Kernel Memory from User Space

## 1. Introduction
오늘날의 OS의 주요한 보안 feature중 하나는 memory isolation. 기본적으로 서로 다른 program이 kernel이나 서로의 memory를 읽을 수 없게 함. => cloud내 한 기기에서 여러 사용자의 여러 프로그램이 동시에 동작할 수 있음.<br>
[Page table entry](https://wiki.osdev.org/Paging)에 [user/supervisor bit](https://stackoverflow.com/questions/30089287/what-does-the-user-supervisor-bit-in-the-page-table-entry-mean)를 둬서 supervisor만 kernel을 접근할 수 있도록 함.
추가적으로 cpu내의 [user/supervisor register](https://en.wikipedia.org/wiki/Status_register)가 존재해 현재 어떤 권한으로 operation이 실행되는지 나타냄.<br>
이 동작은 user/supervisor bit가 kernel로 들어갈때만 set되고, user process로 나올땐 clear될꺼라는 ground truth위에서 작동.
이를 통해서 모든 프로그램의 address space상에 kernel이 mapping될 수 있게 되고, user, supervisor사이의 효율적인 transition이 가능. => user process에서 kernel로 넘어갈 때 memory mapping의 변화가 발생하지 않음.<br>
Meltdown => user process가 kernel영역을 다 읽어버리는 공격.
Meltdown은 software의 취약점이 아님. CPU의 기능을 이용하는 하드웨어레벨의 공격.
취약한 CPU에서 code를 동작할 수 있는 능력만 있으면 걍 kernel address space전체를 dump떠버릴 수 있다고 한다.<br>
Meltdown의 주요 원인은 Out of order execution(OoOE). 
OoOE의 문제점이, privilege없는 값을 가지고도 미리 연산을 때려버린다는 것이다. 그러나, 잘못된 연산이라는 것이 드러나면, CPU는 자체적으로 코드 실행 flow를 수정한다. 그러나 여기서 문제점은 cache가 영향을 받는다는 것이다.

## 2. Background
### 2.1. Out of Order Execution
CPU의 utilization을 극대화하기 위해서 code를 순차적으로 실행하지 않는 것.<br>
Tomasulo가 1967년에 알고리즘 제안. Unified reservation station을 도입. RAW, WAR, WAW해저드를 막기 위해 register rename수행. Common data bus(CDB)로 모든 reservation unit들을 execution unit들과 연결한다. 필요한 operand가 준비 안되었으면, reservation unit은 CDB에 listen을 한다. listen하다가 필요한 operand가 준비되면 바로 실행한다.<br>
x86 intel cpu의 frontend에서 memory로부터 instruction을 fetch해오고 micro opcode로 decode한 후에 execution engine으로 전달한다. Reorder buffer에서 register allocation, register renaming, retiring 등등 최적화 및 여러 기능 수행. Scheduler(Unified Reservation Station)에서 reorder buffer로부터 받아온 micro opcode를 Execution unit들과 연결되어있는 exit port들로 넣는다. Execution unit들이 각각 수행한다. (superscalar)<br>
branch predictor는 실제로 branch가 정해지기 전에 미리 예측 때리는 것. Branch와 연관된 code가 현재 실행하는 것에 의존성을 가지지 않는다면 미리 실행해볼 수 있다. (speculative execution)
미리 실행하다가 예측이 맞으면 계속 미리 실행하던거 쓰면 되고, 예측 틀리면 reorder buffer랑 scheduler 비우고 다시 초기화 해서 정상루틴으로 돌아온다.

### 2.2 Address Space
각각의 process마다 virtual address space가지고 있어서 memory isolation이 이루어짐.<br>
각각의 virtual address space는 user영역과 kernel영역이 존재. kernel영역에는 kernel이 사용하는 데이터를 관리하는 것 뿐만 아니라, user영역을 수정해야할 때를 위해서 physical memory 전체가 mapping되어있는 영역이 존재한다고 한다. Linux, OS X에서는 direct-physical map을 이용해 전체 physical memory를 pre-define된 virtual address에다가 넣는다. Windows에서는 [paged pools, non-paged pools, system cache](https://superuser.com/questions/1410289/what-are-commited-memory-cached-paged-not-paged-pool-how-they-are-d)가 존재. Pool들은 physical memory를 virtual memory공간으로 mapping한 것인데, 페이징 파일에 기록되거나, 불려올 수 있는 가상 메모리 공간인 paged pools, 실제 메모리 상에 상주하여 접근 가능한 non-paged pools 존재. Kernel cache는 추가적으로 모든 file-backed page들(file을 읽어들여서 만든 page)에 대한 mapping도 가지고 있다.<br>
Kernel ASLR(KASLR)을 이용해 부팅할때마다 driver들의 offset을 바꿔서 공격 힘들게 해왔는데, meltdown으로 우회 가능.

### 2.3. cache attacks
#### Cache side-channel attacks
Cache 도입으로 인한 데이터 접근 시간의 차이를 이용한 공격.<br>
Evict + time: victim이 접근하는 memory영역을 확인하기 위한 기법. Victim이 접근하는 memory 영역을 모두 cache에서 제거하고, victim을 실행하고 걸린시간 측정.<br>
Prime + probe: 공격자가 cache를 기준값(prime)으로 채워두고 victim을 실행. Victim이 어떤 데이터를 cache에 채운다면, 공격자가 기존에 올려놨던 prime이 evict된다. 공격자가 다시 cache를 읽었을 때 victim이 사용한 cache부분을 접근할 때 접근시간이 느린 것을 이용해 victim이 접근한 메모리의 위치를 알 수 있다.<br>
#### Flush + reload
Attacker가 캐시를 비우고, victim이 접근한 곳이 캐시에 채워지면, attacker가 같은 곳 접근할 때 해당부분이 더 빠른 것을 이용.

## 3. Toy example
array접근하는 코드 위에 exception을 발생시키는 코드를 넣음. 원래는 array접근 위에 exception이 발생하기 때문에 array접근이 이론상 안되어야 하지만, OoOE때문에 접근이 된다. 하지만 잘못된 예측이기 때문에 결과값이 memory나 register로 commit되진 않지만, 계산과정에서 cache에 해당 값이 올라오기 때문에 flush reload같은 cache side channel attack을 이용해서 값을 읽을 수 있다. flush reload말고 여러가지 더 공격기법 있는데 flush reload가 가장 정확하고 간단한걸로 알고있어서 이거로 뚫어봄.<br>
access(probe_array[data * 4096])에서 probe_array는 char[]형이기 때문에 data값에 따른 array접근 위치는 한 page당 하나밖에 없다. 4096을 곱했기 때문. 이렇게 서로 다른 page에 접근하는 영역을 흩뿌려 놓으면, prefetcher에 의한 false positive도 막을 수 있다. prefetcher가 prefetch를 서로 다른 page에 대해서 수행하진 않기 때문이다.<br>
이제 다시 data값을 0부터 하나하나 차례대로 넣어보면서 page 접근 속도를 비교했을 때 접근 속도가 빠른 data값을 알아내면 된다.

## 4. Building Blocks of the Attack
공격은 두 building block으로 이루어져 있다. 첫번째 building block에서 OoOE를 이용해 secret을 가지고 연산을 한다. OoOE를 통한 side effect를 남기는 instruction을 transient instruction이라고 하자. 두번째 building block에서는 첫번째 building block에서 남긴 side effect를 이용해서 secret을 leak해오는 부분이다.

### 4.1 Executing Transient Instructions
Meltdown building block중에 첫번째 block에서 transient instruction을 실행한다. 근데 CPU특성상 거의 모든 code가 transient instruction이다. 계속 OoOE를 통한 최적화를 진행하니까. 따라서 secret을 operand로 사용하는 transient instruction을 찾으면 된다.<br>
attacker의 process에 mapping되어있는 주소값들에 집중하기로 했다. 접근가능한 주소들이나 아니면 kernel부분같이 접근 불가능한 주소 말이다. 다른 process에 있는 다른 주소공간에 대해서도 공격할 수 있으나, 어차피 kernel에 physical memory가 mapping되어있는 곳이 있기 때문에 그부분은 논외로 한다.<br>
근데 문제점이, 일반 user가 kernel영역 접근하면 exception 걸리면서 프로세스가 종료된다. 그래서 이걸 핸들링 해야한다. Exception handling, Exception suppression 두 방법을 제시한다.
#### Exception handling
invalid memory location을 접근하기 전에, fork를 떠놓고, child process를 이용해 접근하고 죽이는 방법이 있다. child process가 secret가지고 연산하고 죽으면, parent process에서 side channel attack을 통해 secret을 알아내는 방식이다. 아니면 segmentation fault와 같은 signal에 대한 handler를 공격자가 설치해서 application이 crash나는 것을 방지할 수 있다. 이를 이용하면 새 process를 만드는 overhead를 없앨 수 있다.
#### Exception suppression
exception이 맨 첫번째로 동작하지 못하도록 하는 것이다. [transactional memory](https://ko.wikipedia.org/wiki/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%94%EB%84%90_%EB%A9%94%EB%AA%A8%EB%A6%AC)를 이용하는 방법이다.<br>
memory 접근을 그룹화하여 atomic하게 실행하는 것인데, error가 났을 때 previous state로 roll back하는 option을 줘서 exception이 발생해도 previous state로 roll back되고, 프로그램이 중단되지 않고 계속 진행되게 할 수 있다.<br>
추가적으로, execution code path에 존재하지 않는 코드가 speculative execution에 의해서 실행될 수 있다. 그런데 그러한 code흐름은 해당 branch를 이전에 실행했었을 때 true가 되는 등 어쨌든 branch predictor를 학습시켜야 한다. 따라서 invalid memory access가 speculative execute되지 않게 하기 위해서 branch predictor를 잘 학습시켜야 한다.

### 4.2. Building a Convert Channel
두번째 building block에서는 첫번째 building block의 side effect를 이용해서 값을 가져오는 역할을 한다. 값을 받는 receiver는 반드시 transient instruction sequence의 한 부분이어야 하는 것은 아니다. 다른 thread 심지어는 다른 process에서도 receiver역할을 할 수 있다.(그래서 Exception handling기법이 가능)<br>
Flush reload가 빠르고 noise가 적기 때문에 이걸 사용. monitor하는 cache에 속한 address를 접근할 때 sender가 bit 1을 transmit하고, 아닐경우엔 bit 0을 전송할 수 있다. 서로 다른 여러 cache line을 이용하면 한번에 1bit씩 말고도 여러 bit를 한꺼번에 전달할 수 있다. 그런데 flush reload가 오래걸리는 공격이라서 그냥 1bit씩 전송하고 위치에 맞게 shift와 masking을 하는게 효율적이라고 한다.<br>
cache말고도 어떠한 side effect를 남기는 부분이 있다면 공격에 알아서 잘 써보세요.

## 5. Meltdown
#### Attack Setting
personal computer또는 cloud에서 공격하는 상황.<br>
unprivileged code 아무거나 실행 가능한 상황.<br>
공격자가 physical memory에는 접근 못하는 상황.<br>
System은 ASLR, KASLR, NX, SMAP, SMEP, PXN과 같은 mitigation이 적용되어있고, OS는 bug free하다고 가정한다.<br>
PXN: ARM 하드웨어가 제공. SMEP와 비슷.<br>
Attacker의 target은 password나 private key같은 소중한 정보들이다.

### 5.1 Attack description
Step 1: attack의 target이지만 attacker가 접근할 수 없는 memory의 content가 register에 load된다. -> caching<br>
Step 2: transient instruction이 register의 secret content를 기준으로 cache line을 접근한다.<br>
Step 3: Flush reload를 이용해 cache line을 접근하고 값 알아낸다.<br>
위 step을 다양한 memory 주소를 가지고 반복하면 공격자는 전체 physical memory를 포함한 kernel 전체를 dump할 수 있다.<br>
#### Step 1: reading the secret.
memory에 있는 data를 register로 load하기 위해선 virtual address를 사용해야함. virtual address를 physical address로 translate함과 동시에 CPU는 virtual address의 permission bit를 체크함. kernel꺼 읽으려고 하면 kernel영역이 모든 user의 virtual address space에 load되어있기 때문에 physical memory address로 translate가 잘 되는데, permission bit때문에 exception이 발생할 뿐이다.<br>
Listing 2에 있는 code가 uOP로 바뀌어서 OoOE가 진행됨. uOP는 실행이 끝나면 in-order로 retirement되고 결과는 commit된다. 근데 line 4의 MOV가 실행되고 retire될 때 exception이 발생해서 모든 subsequent instruction의 결과를 버리기 위해 OoOE되던 pipeline이 flush된다. 그러나 exception을 raising하는 것과 data load코드가 실행되는 것 사이에 race condition이 발생한다.
#### Step 2: transmitting the secret
MOV instruction이 retire되기 전에 probe array를 접근하는 code가 OoOE된다.<br>
probe array는 cache에 존재하지 않는 상태여야 한다. 1byte값을 4096과 곱하고 probe array에 접근한다. 4096을 곱함으로써 probe array를 sparse하게 접근하여 cache에 접근하지 않은 다른 부분이 preload되는 것을 막을 수 있다. 한번에 1 byte씩 접근하기 때문에 probe array의 길이는 256 * 4096 byte이다. (4KB page)<br>
exception raising과 race를 하기 때문에 step 2의 runtime을 줄이는게 성능향상에 큰 영향을 준다. 따라서 probe array 접근 속도를 향상시키기 위해 probe array의 주소값을 Translation Lookaside Buffer에 cache시키는 것이 좋다.
#### Step 3: receiving the secret
probe array를 다시 돌아다녀보면서 256page중에 빠르게 접근한 page를 찾으면 secret value를 알 수 있다.
#### Dumping the entire physical memory
exception handling하고, kernel에 physical memory가 mapping된 곳을 계속해서 접근하면 dump가능.

### 5.2 Optimizations and Limitations
#### Inherent bias towards 0
이게 읽는 instruction과 exception handling사이의 race condition으로 읽는 것이기 때문에 읽기 실패할 때가 발생한다. 읽기 실패했을 때 반환되는 값이 0이다. 근데 이게 문제점이 실제 값이 0일때와 구별이 잘 안된다는 것이다. 또한, 필요한 data가 load되지 않아 stall이 되어서 재빠르게 읽어내지 못하는 경우도 발생한다. 따라서 error가 발생할 확률을 줄이기 위해서 읽는 부분에 loop을 넣어서 만약 읽은 값이 0이라면 일정횟수만큼 다시 읽도록 한다.<br>
optimize안했을 때 0의 비율 5.25%, retry loop을 이용하면 0.67%, Intel TSX(Transactional Synchronization Extensions)을 사용하면 0.008%까지 감소.
#### Optimizing the case of 0
어차피 0일때는 공격자가 cache timing 봐도 이게 정상값인지 오류값인지 모르기 때문에 0에 대해서는 cache timing을 재지 않는다. 0이 아닌 나머지부분 다 읽어보고 값을 못찾으면 0이라고 여긴다. 0이 나올 확률을 줄이기 위해서 loop를 돌렸었고, loop를 돌려도 값을 읽는데 성공해서 loop문을 빠져나오는 부분과 exception이 raise되어 loop문을 빠져나오는 부분이 서로 독립적이라 loop문이 공격속도를 유의미하게 느려지게 하지 않는다. 따라서 이러한 optimization을 통해 공격 속도를 증가시킬 수 있다.
#### Single-bit transmission
Transient code에서 한번에 얼마나 긴 data를 줄지는 자유다. 어차피 data의 크기에 따른 code 처리 속도는 비슷하기 때문이다. 그러나, 한번에 작은 bit를 전송해야하는 이유는, 많이 보내면 flush reload check에서 시간을 오지게 잡아먹기 때문이다. 1bit씩 보내고 확인하면 cacheline 하나만 확인하면 된다. 0 bias를 잘 생각해야하는데, 앞에서 error handling을 했기 때문에 오류가 거의 안났다.
#### Exception Suppression using Intel TSX 
TSX를 이용하면 invalid memory access로 인한 exception을 완전히 억누를 수 있다. TSX를 가지고 여러 instruction을 그룹화 해서 atomic한 형태로 만들 수 있다. 모두 실행하거나, 모두 실행 못하거나 두가지이기 때문에, Group 내의 한 instruction이 실패하면, 성공했던 instruction도 revert된다. 그리고 exception은 발생하지 않는다.
#### Dealing with KASLR
Kernel Address Space Layout Randomization. Boot time에 kernel space를 random화 한다. Physical memory가 mapping되어있는 곳도 random화 되기 때문에 공격자가 offset 잘 찾아야 함. [근데 RAM이 8기가일때, 최대 128번정도만 test하면 offset을 찾을 수 있다고 한다.](https://security.stackexchange.com/questions/178731/how-did-the-meltdown-attack-break-kaslr-in-128-steps-for-a-target-machine-with-8) CPU가 사용하는 address space bit 40bit. 8기가를 모두 kernel memory에 mapping했으니 33bit사용. 따라서 나머지 7bit만 잘 조정하면 된다고 한다.

## 6. Evaluation
Lab, Cloud, Phone에서 돌려봄. Amazon ECC를 썼는데, Cloud에서는 윤리적인 문제 땜시 다른 사람이 사용하는 부분은 보지 않았다고 한다.

### 6.1. Leakage and Environments
Window 10, Linux, android에서 해봄. KAISER 적용시 meltdown을 막을 수 있다는 것을 보여주기 위해서 KAISER적용후에도 공격해봄.

#### 6.1.1 Linux
KASLR안걸려있을 땐 0xffff880000000000에 physical memory가 mapping되어있다. 그래서 거기 읽으면 된다. KASLR걸려있으면 최대 128번 다시 시도하면 된다.

#### 6.1.2 Linux with KAISER Patch
KAISER는 Kernel전체를 user address space에 mapping하지 않는다. Interrupt handler같은 필요한 일부분만 넣어놓는다. 따라서 physical memory가 mapping되어있는 kernel영역은 user address space에 존재하지 않기 때문에 meltdown이 안먹힌다. User space에 mapping되는 kernel영역도 랜덤으로 mapping되어서 찾기 힘들다.

#### 6.1.3 Microsoft Windows
Windows는 linux처럼 physical memory전체를 linear하게 mapping하진 않지만, paged pool, non-paged pool, systemcache에 physical memory를 mapping해놓는다. 얘도 user address space에 mapping이 되기 때문에 windows kernel debugger로 한번 봐보고 meltdown으로 leak해보고 비교하고 그랬다.

#### 6.1.4 Android
삼성 갤럭시 s7가지고 성공. 0xffffffbfc0000000에 physical memory가 mapping되어있었다고 한다.

#### 6.1.5 Containers
Container도 kernel을 공유하기 때문에 meltdown 때리면 같은 물리적 host에 존재하는 다른 container의 정보도 전부 들여다볼 수 있었다.

#### 6.1.6 Uncached and Uncacheable Memory
보드에 여러 CPU가 있고 다른 CPU의 cache에 target data가 있고 공격자의 CPU의 L1 cache에는 값이 없는 상태에서 공격해도 낮은 읽기 속도로 여전히 공격이 가능했다. 이는 공격자의 CPU의 L1 cache에 target data가 존재해야하는 것이 meltdown의 조건이 아니라는 것을 보여준다.<br>
Uncacheable memory공간은 cache를 우회해서 바로 memory로 넣는다고 한다. 이러한 공간도 한 cpu에서 해당 uncacheable memory공간을 제대로 접근하는 루틴과 meltdown 루틴을 thread로 병렬적으로 돌려버리면, 공격이 가능하다고 한다. thread간에 [line fill buffer](https://community.intel.com/t5/Intel-Moderncode-for-Parallel/What-is-the-aim-of-the-line-fill-buffer/m-p/1180777)가 공유되기 때문에 여기서 data를 가져오는 것으로 여긴다고 한다. 하지만 Memory Mapped IO region은 안된다.

### 6.2 Meltdown Performance
속도 뿐만아니라 error율도 확인해봄.<br>
L1캐시와 같이 core에 가까운데에 데이터가 존재할 때에는 거의 race condition에서 이긴다. I7 8700k로 10번 10초동안 돌려보니까 평균 582kb/s, 오류율 0.003%이 나왔다. i7 6700k, 제온으로도 한 결과 있음. 나중에 ppt 만들때 보고 만들자. 더 느리지만 정확한 버전도 있는데, 평균 137kb/s걸리지만 에러가 0이 나왔다고 한다. L1에 없고 L3에만 target data가 존재한다고 하더라도 느리지만 race condition에서 승리 가능. 그러나 cache에 없으면 race condition 성공하기가 힘듦. 따라서 다른 thread가 target부분을 미리 접근해서 cache에 올려놔야 성능이 빨라진다. 또는 hardware prefetcher를 speculative access를 통해 trigger한다.

### 6.3 Limitations on ARM and AMD
ARM이랑 AMD는 잘 안되더라. AMD는 architecture가 달라서 그런듯. Intel은 uop가 retire할 때 이게 제대로 된건지 확인하는건데 ARM, AMD는 내부 아키텍쳐가 달라서 인텔보다 더 빠르게 exception 날리는 듯. 그러나 ARM, AMD에서도 OoOE를 쓰기때문에 toy example은 먹힘.

## 7. Countermeasures(대책)
1. 하드웨어 아키텍쳐를 바꾼다.
2. KAISER를 적용한다. 원래는 KASLR을 강화하기 위해서 만든건데 이게 우연히 Meltdown도 막을 수 있다.

## 7.1. Hardware
OoOE때문에 meltdown났으니까 OoOE비활성화 하면 된다. 근데 이러면 느려짐.<br>
Permission check는 병렬적으로 진행되지 않도록 하여 막을 수도 있다. 근데 이러면 매번 fetch할때마다 stall걸리니까 또 느려짐.<br>
가장 그나마 나은건 Hard split을 이용해 address space를 반 뚝 잘라서 유저영역과 privileged영역을 구분하는 것. 이렇게 하면 권한을 접근할 때 따로 확인할 필요 없이 주소값만 보고도 접근이 가능한지 아닌지 확인이 가능해서 더 빠르게 exception 낼 수 있다.

## 7.2 KAISER
현재 시중에 굴러다니는 하드웨어는 패치가 불가능하다. 따라서 software적으로 패치를 해야한다.<br>
KAISER를 이용해 user address space에 kernel이 최소한만 올라가서 physical memory mapped page이런게 접근 불가능하도록 해서 meltdown을 막는다.<br>
그러나, x86 architecture design때문에 kernel의 일부 instruction은 여전히 user address space에 있어야 해서 이게 나중에 meltdown의 attack surface로 사용될 수 있다.<br>
그러나 지금 한시가 급한 상황에서 KAISER는 가장 좋은 방법이므로 이 패치를 모두 적용시킨다.<br>
User space에 있는 kernel pointer를 이용해서 KASLR을 뚫는 것을 막기 위해서 user space에 있는 pointer는 kernel function으로 바로 가지 않고 kernel 내에 [trampoline function](https://en.wikipedia.org/wiki/Trampoline_(computing))으로 가게 된다. 그리고 trampoline function에서 random offset을 이용해 kernel내부에 진짜 routine으로 가는 것이다. 한번 indirect jump 거치는 듯.<br>
Linux에는 [KPTI(Kernel Page Table Isolation)](https://en.wikipedia.org/wiki/Kernel_page-table_isolation)로, Windows에는 [KVA Shadow](https://msrc-blog.microsoft.com/2018/03/23/kva-shadow-mitigating-meltdown-on-windows/)로써, iOS에는 [Double Map](https://twitter.com/aionescu/status/948609809540046849)으로써 적용되었다.
