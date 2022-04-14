---
sort: 6
---

# Vim

## vimrc

```
set ts=4
set sw=4
set sts=4
set showmatch
set number
set hlsearch
set incsearch
set ai
set expandtab

filetype plugin indent on
autocmd FileType python setlocal tabstop=4 expandtab shiftwidth=4 softtabstop=4
if has("syntax")
    syntax on
endif
```

## vim 명령어

vs: vertical split<br>
sp: (horizontal) split
