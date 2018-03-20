---
title: vim备忘录
categories: 备忘录
tags:
  - vim
---

# C语言VIM设置
```
:set nu "设置显示行号  
:set bg=dark "背景色设置  
:set cindent "设置c语言自动对齐  
:set history=1000 "设置历史记录条数  
:set ts=4	"设置tab对应空格数
:set expandtab "vim 里面按下tab 会自动替换为4个空格
```

# tab和空格替换
```
TAB替换为空格：
:set ts=4
:set expandtab
:%retab!

空格替换为TAB：
:set ts=4
:set noexpandtab
:%retab!
```

# VIM代码格式化
## astyle
astyle是款代码格式化工具，具体使用参考官网：http://astyle.sourceforge.net/

## astyle和vim整合
在和vim整合之前，首先应保证astyle成功安装，并可用。
vim具有掉外部命令的能力，因此可以在.vimrc中添加如下代码，编辑完成后，按F2即可对代码进行格式。
```
map <F2> :call FormatCode()<CR>
func! FormatCode()
    exec "w"
    if &filetype == 'c' || &filetype == 'h'
        exec "!astyle --style=linux --suffix=none %"
    elseif &filetype == 'cpp' || &filetype == 'cc' || &filetype == 'hpp'
        exec "!astyle --style=linux --suffix=none %"
    elseif &filetype == 'perl'
        exec "!astyle --style=gnu --suffix=none %"
    elseif &filetype == 'py'|| &filetype == 'python'
        exec "!autopep8 --in-place --aggressive %"
    elseif &filetype == 'java'
        exec "!astyle --style=java --suffix=none %"
    elseif &filetype == 'jsp'
        exec "!astyle --style=gnu --suffix=none %"
    elseif &filetype == 'xml'
        exec "!astyle --style=gnu --suffix=none %"
    else
        exec "normal gg=G"
        return
    endif
endfunc
```