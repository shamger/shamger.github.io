+++
date = '2025-08-17T17:56:16+08:00'
draft = false
title = '新mac配置备忘'
+++

### mac终端配置
使用 iterm2+zsh+oh-my-zsh
- mac默认使用的是zsh作为shell，可以echo $SHEL确认
- 安装iterm2，官网下载
- 安装oh-my-zsh：sh -c "$(curl -fsSL raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

### vim配置
不考虑使用花里胡俏的插件，只使用以下简短几行配置，让vim符合自己使用习惯即可。
1. 打开～/.vimrc
2. 添加以下配置
```
set nu
sy on
set ruler
set smartindent shiftwidth=4
set tabstop=4
set expandtab
colorscheme desert
```
3.设置 Vim 内部使用的字符编码
```
" 设置 Vim 内部使用的字符编码
set encoding=utf-8

" 设置自动识别的文件编码列表（按优先级从高到低）
" 这样 Vim 会依次尝试 utf-8, gbk 等，解决大部分中文乱码
set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936

" 设置当前终端使用的编码
set termencoding=utf-8

" 解决菜单、提示信息乱码（如果有的话）
language message zh_CN.UTF-8
```
