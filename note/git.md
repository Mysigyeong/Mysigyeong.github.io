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

rsa 키쌍을 생성하고, ssh에 등록한다.

```
ssh-keygen -t rsa -b 4096 -C "email"
eval `ssh-agent -s`
ssh-add (private key)
```

public key는 github 사이트에 들어가 등록한다.