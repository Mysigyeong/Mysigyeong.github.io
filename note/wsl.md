---
sort: 7
---

# wsl

## GUI
windows 11 빌드 22000이상부턴 기본적으로 wsl GUI기능을 지원한다.<br><br>
VcXsrv를 설치<br>
VcXsrv에서 Display settings에서 Display number 0으로 설정<br>
Extra settings에서 Disable access control 또는 추가 인수로 -ac 입력<br><br>
bashrc에는 다음 입력
```
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0
```

## 한글 깨짐
```
sudo apt install fonts-nanum*
```