---
sort: 7
---

# input

home 디렉토리에 input.c 파일을 통해 소스코드를 볼 수 있다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
        printf("Welcome to pwnable.kr\n");
        printf("Let's see if you know how to give input to program\n");
        printf("Just give me correct inputs then you will get the flag :)\n");

        // argv
        if(argc != 100) return 0;
        if(strcmp(argv['A'],"\x00")) return 0;
        if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
        printf("Stage 1 clear!\n");

        // stdio
        char buf[4];
        read(0, buf, 4);
        if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
        read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
        printf("Stage 2 clear!\n");

        // env
        if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
        printf("Stage 3 clear!\n");

        // file
        FILE* fp = fopen("\x0a", "r");
        if(!fp) return 0;
        if( fread(buf, 4, 1, fp)!=1 ) return 0;
        if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
        fclose(fp);
        printf("Stage 4 clear!\n");

        // network
        int sd, cd;
        struct sockaddr_in saddr, caddr;
        sd = socket(AF_INET, SOCK_STREAM, 0);
        if(sd == -1){
                printf("socket error, tell admin\n");
                return 0;
        }
        saddr.sin_family = AF_INET;
        saddr.sin_addr.s_addr = INADDR_ANY;
        saddr.sin_port = htons( atoi(argv['C']) );
        if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
                printf("bind error, use another port\n");
                return 1;
        }
        listen(sd, 1);
        int c = sizeof(struct sockaddr_in);
        cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
        if(cd < 0){
                printf("accept error, tell admin\n");
                return 0;
        }
        if( recv(cd, buf, 4, 0) != 4 ) return 0;
        if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
        printf("Stage 5 clear!\n");

        // here's your flag
        system("/bin/cat flag");
        return 0;
}
```

python pwntools를 사용하면 풀 수 있다.<br>
home 디렉토리 내부에서는 파일을 생성할 수 없기 때문에 /tmp 폴더 내부에 폴더 하나 생성하고 그 속에서 진행하자.

```python
# input.py

from pwn import *;
import socket;

# stage 1
argv_list = [str(i) for i in range(100)]
argv_list[65] = '\x00'
argv_list[66] = '\x20\x0a\x0d'
argv_list[67] = '10000'

# stage 3
env_dict = {'\xde\xad\xbe\xef':'\xca\xfe\xba\xbe'}

# stage 2
with open('./stderr_file', 'a') as f:
        f.write('\x00\x0a\x02\xff')

# stage 4
with open('./\x0a', 'a') as f:
        f.write('\x00\x00\x00\x00')

p = process(argv=argv_list, executable='/home/input2/input', env=env_dict, stderr=open('./stderr_file'))

"""
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
"""
print(p.recvline())
print(p.recvline())
print(p.recvline())
print(p.recvline())

# stage 2
p.send('\x00\x0a\x00\xff')

"""
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
"""
print(p.recvline())
print(p.recvline())
print(p.recvline())

# stage 5
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client_socket.connect(('127.0.0.1', 10000))
client_socket.send('\xde\xad\xbe\xef')
client_socket.close()

p.interactive()
```

그러나 여기서 문제가 발생한다. 현재 디렉토리가 home 디렉토리가 아니기 때문에 home 디렉토리에 있는 flag를 실행할 수 없게 된다.<br>
따라서 심볼릭 링크를 이용하여 /tmp 하위 디렉토리 내부에서도 home 디렉토리에 있는 flag를 읽을 수 있도록 해야한다.

```bash
input2@pwnable:/tmp/dlfjs3$ ln -s /home/input2/flag flag
input2@pwnable:/tmp/dlfjs3$ python input.py
```

```tip
## 알게 된 점

pwntools
```