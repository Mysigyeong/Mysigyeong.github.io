---
sort: 1
---

# docker

## 디버깅, X11 forwarding 활성화 및 wsl docker gui forwarding

Mobaxterm으로 wsl접속하는게 마음 편하다.<br>
wsl에서 DISPLAY 환경변수를 export 해주어야한다.

```
export DISPLAY="`grep nameserver /etc/resolv.conf | sed 's/nameserver //'`:0.0"

docker run -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix --cap-add=SYS_PTRACE --security-opt seccomp=unconfined --security-opt apparmor=unconfined -it image:tag /bin/bash
```

## docker commit 및 push

image_name은 `dockerhub_id/repository:tag`로 해야한다.

```bash
# docker commit
docker commit (container) (image_name)

# docker login
docker login

# docker push
docker push (image_name)
```

## docker image 이름 변경

```
docker image tag (old_image_name) (new_image_name)
docker rmi (old_image_name)
```