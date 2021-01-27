---
sort: 2
---

# Git

## github 파일 삭제

```
git rm -rf (file)
git commit -m "commit message"
git push origin (branch)
```

## github ssh key 등록

rsa 키쌍을 생성하고, ssh-agent에 등록한다.

```
ssh-keygen -t rsa -b 4096 -C "email"
eval `ssh-agent -s`
ssh-add (private key)
```

public key는 github 사이트에 들어가 등록한다.<br>
ssh-agent를 실행하고 ssh-add를 통해 key를 등록하는 것은 해당 key를 이용하기 전에 계속 해줘야한다.<br>
귀찮아서 bashrc파일에 넣어놨다.<br><br>
ssh연결을 이용하기 위해서는 https로 설정되어있는 origin을 ssh로 바꿔야한다.

```
git remote remove origin
git remote add origin (저장소 ssh주소)
git remote show
```