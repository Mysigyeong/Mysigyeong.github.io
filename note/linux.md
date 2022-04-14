---
sort: 5
---

# Linux

## 버전 확인

```shell
uname -a
cat /etc/issue
lsb_release -a
```

## 듀얼부팅 grub 자동부팅 OS 선택

```shell
sudo vi /etc/default/grub

# 파일 내부
GRUB_DEFAULT=0 # 부팅시 뜨는 선택창 맨 윗줄부터 0, 1, 2
GRUB_TIMEOUT=10 # 자동 부팅 대기시간

# 변경 후 저장
sudo update-grub
```

## nvidia 그래픽카드 드라이버 설치
Software & Updates에 Additional Drivers에서 설치하면 된다.

## .deb 파일 설치
```
sudo dpkg -i package.deb
```