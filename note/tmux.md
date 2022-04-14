---
sort: 3
---

# tmux and terminal

## tmux

### 단축키 ctrl b를 ctrl a로 변경

home 디렉토리에다가 `.tmux.conf`파일 생성 후 아래 입력.

```
unbind C-b
set-option -g prefix C-a
bind-key C-a send-prefix
```

## terminal

Alt + Shift + '+' : 세로로 화면분할<br>
Alt + Shift + '-' : 가로로 화면분할
