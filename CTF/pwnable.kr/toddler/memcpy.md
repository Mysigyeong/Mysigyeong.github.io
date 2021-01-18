---
sort: 17
---

# memcpy

home 디렉토리에 memcpy.c파일에 소스코드가 들어있다.

```c
// compiled with : gcc -o memcpy memcpy.c -m32 -lm
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <math.h>

unsigned long long rdtsc(){
        asm("rdtsc");
}

char* slow_memcpy(char* dest, const char* src, size_t len){
        int i;
        for (i=0; i<len; i++) {
                dest[i] = src[i];
        }
        return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
        size_t i;
        // 64-byte block fast copy
        if(len >= 64){
                i = len / 64;
                len &= (64-1);
                while(i-- > 0){
                        __asm__ __volatile__ (
                        "movdqa (%0), %%xmm0\n"
                        "movdqa 16(%0), %%xmm1\n"
                        "movdqa 32(%0), %%xmm2\n"
                        "movdqa 48(%0), %%xmm3\n"
                        "movntps %%xmm0, (%1)\n"
                        "movntps %%xmm1, 16(%1)\n"
                        "movntps %%xmm2, 32(%1)\n"
                        "movntps %%xmm3, 48(%1)\n"
                        ::"r"(src),"r"(dest):"memory");
                        dest += 64;
                        src += 64;
                }
        }

        // byte-to-byte slow copy
        if(len) slow_memcpy(dest, src, len);
        return dest;
}

int main(void){

        setvbuf(stdout, 0, _IONBF, 0);
        setvbuf(stdin, 0, _IOLBF, 0);

        printf("Hey, I have a boring assignment for CS class.. :(\n");
        printf("The assignment is simple.\n");

        printf("-----------------------------------------------------\n");
        printf("- What is the best implementation of memcpy?        -\n");
        printf("- 1. implement your own slow/fast version of memcpy -\n");
        printf("- 2. compare them with various size of data         -\n");
        printf("- 3. conclude your experiment and submit report     -\n");
        printf("-----------------------------------------------------\n");

        printf("This time, just help me out with my experiment and get flag\n");
        printf("No fancy hacking, I promise :D\n");

        unsigned long long t1, t2;
        int e;
        char* src;
        char* dest;
        unsigned int low, high;
        unsigned int size;
        // allocate memory
        char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
        char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
        src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

        size_t sizes[10];
        int i=0;

        // setup experiment parameters
        for(e=4; e<14; e++){    // 2^13 = 8K
                low = pow(2,e-1);
                high = pow(2,e);
                printf("specify the memcpy amount between %d ~ %d : ", low, high);
                scanf("%d", &size);
                if( size < low || size > high ){
                        printf("don't mess with the experiment.\n");
                        exit(0);
                }
                sizes[i++] = size;
        }

        sleep(1);
        printf("ok, lets run the experiment with your configuration\n");
        sleep(1);

        // run experiment
        for(i=0; i<10; i++){
                size = sizes[i];
                printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
                dest = malloc( size );

                memcpy(cache1, cache2, 0x4000);         // to eliminate cache effect
                t1 = rdtsc();
                slow_memcpy(dest, src, size);           // byte-to-byte memcpy
                t2 = rdtsc();
                printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

                memcpy(cache1, cache2, 0x4000);         // to eliminate cache effect
                t1 = rdtsc();
                fast_memcpy(dest, src, size);           // block-to-block memcpy
                t2 = rdtsc();
                printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
                printf("\n");
        }

        printf("thanks for helping my experiment!\n");
        printf("flag : ----- erased in this source code -----\n");
        return 0;
}
```
memcpy를 구현한 코드다. fast_memcpy에서는 한번에 16byte씩 복사하는 것을 확인할 수 있다.<br>
무엇이 문제인지 확인하기 위해 직접 빌드한 후 돌려보았다.

```
memcpy@pwnable:/tmp/dlfjs123123$ gcc -o memcpy /home/memcpy/memcpy.c -m32 -lm
memcpy@pwnable:/tmp/dlfjs123123$ gdb memcpy
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1

... (중략) ...

Reading symbols from memcpy...(no debugging symbols found)...done.
(gdb) run
Starting program: /tmp/dlfjs123123/memcpy
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 16
specify the memcpy amount between 32 ~ 64 : 32
specify the memcpy amount between 64 ~ 128 : 64
specify the memcpy amount between 128 ~ 256 : 128
specify the memcpy amount between 256 ~ 512 : 256
specify the memcpy amount between 512 ~ 1024 : 512
specify the memcpy amount between 1024 ~ 2048 : 1024
specify the memcpy amount between 2048 ~ 4096 : 2048
specify the memcpy amount between 4096 ~ 8192 : 4096
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 1976
ellapsed CPU cycles for fast_memcpy : 238

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 230
ellapsed CPU cycles for fast_memcpy : 268

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 288
ellapsed CPU cycles for fast_memcpy : 398

experiment 4 : memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy : 482
ellapsed CPU cycles for fast_memcpy : 122

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 1124

Program received signal SIGSEGV, Segmentation fault.
0x080487cc in fast_memcpy ()
(gdb) disas
Dump of assembler code for function fast_memcpy:
   0x08048798 <+0>:     push   %ebp

   ... (중략) ...

   0x080487b6 <+30>:    mov    0x8(%ebp),%edx
   0x080487b9 <+33>:    movdqa (%eax),%xmm0
   0x080487bd <+37>:    movdqa 0x10(%eax),%xmm1
   0x080487c2 <+42>:    movdqa 0x20(%eax),%xmm2
   0x080487c7 <+47>:    movdqa 0x30(%eax),%xmm3
=> 0x080487cc <+52>:    movntps %xmm0,(%edx)
   0x080487cf <+55>:    movntps %xmm1,0x10(%edx)
   0x080487d3 <+59>:    movntps %xmm2,0x20(%edx)
   0x080487d7 <+63>:    movntps %xmm3,0x30(%edx)
```

fast_memcpy에서 movntps를 이용해서 메모리에 레지스터값을 옮길 때 segmentation fault가 발생하는 것을 알 수 있다.<br>
딱히 뭐 문제되는거 같아보이는게 없는데 일단 여기에 break point를 걸고 다시 살펴보기로 했다.

```gdb
(gdb) b* 0x080487cc
Breakpoint 1 at 0x80487cc
(gdb) r
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /tmp/dlfjs123123/memcpy
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 16
specify the memcpy amount between 32 ~ 64 : 32
specify the memcpy amount between 64 ~ 128 : 64
specify the memcpy amount between 128 ~ 256 : 128
specify the memcpy amount between 256 ~ 512 : 256
specify the memcpy amount between 512 ~ 1024 : 512
specify the memcpy amount between 1024 ~ 2048 : 1024
specify the memcpy amount between 2048 ~ 4096 : 2048
specify the memcpy amount between 4096 ~ 8192 : 4096
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 1960
ellapsed CPU cycles for fast_memcpy : 216

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 192
ellapsed CPU cycles for fast_memcpy : 256

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 278
ellapsed CPU cycles for fast_memcpy : 378

experiment 4 : memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy : 492

Breakpoint 1, 0x080487cc in fast_memcpy ()
(gdb) i r edx
edx            0x804c460        134530144
(gdb) c
Continuing.
ellapsed CPU cycles for fast_memcpy : 26440435572

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 944

Breakpoint 1, 0x080487cc in fast_memcpy ()
(gdb) i r edx
edx            0x804c4a8        134530216
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x080487cc in fast_memcpy ()
```

edx값이 0으로 끝날 때는 정상적으로 들어가는데, edx값이 8로 끝날 때에 segmentation fault가 뜬다.<br>
memory alignment를 맞춰줘야하는 것으로 보인다. 따라서 alignment를 맞출 수 있도록 memory크기를 조절해야한다.

```bash
memcpy@pwnable:~$ nc 0 9022
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 24
specify the memcpy amount between 32 ~ 64 : 40
specify the memcpy amount between 64 ~ 128 : 72
specify the memcpy amount between 128 ~ 256 : 136
specify the memcpy amount between 256 ~ 512 : 264
specify the memcpy amount between 512 ~ 1024 : 520
specify the memcpy amount between 1024 ~ 2048 : 1032
specify the memcpy amount between 2048 ~ 4096 : 2056
specify the memcpy amount between 4096 ~ 8192 : 4104
ok, lets run the experiment with your configuration
```

```tip
## 알게 된 점

memory alignment
```