---
sort: 4
---

# flag

flag 바이너리를 다운받을 수 있다.

<img src="/picture/pwnable.kr/flag_1.png" width="1000"/>

디스어셈블이 제대로 되지 않아보여서 맨 첫부분을 봤더니 UPX packing이 되어있었다.<br>
packing이란, reverse engineering을 쉽게 하지 못하도록 하는 일종의 난독화라고 생각하면 된다.<br>
upx packing 도구는 [여기](https://upx.github.io/)서 다운받을 수 있다.

```bash
$ ./upx -d ./flag -o flag_decompressed
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2020
UPX 3.96        Markus Oberhumer, Laszlo Molnar & John Reiser   Jan 23rd 2020

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    883745 <-    335288   37.94%   linux/amd64   flag_decompressed

Unpacked 1 file.
```

unpacking하고 main함수 들여다보자마자 flag변수가 보인다.

<img src="/picture/pwnable.kr/flag_2.png" width="1000"/>

```tip
## 알게 된 점

packing
```