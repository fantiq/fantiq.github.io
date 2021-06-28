---
title: vim的使用以及配置
tags: [vim]
categories: [vim]
date: 2016-06-15 15:37:54
thumbnailImage: icon.jpg
thumbnailImagePosition: right
autoThumbnailImage: yes
coverImage: cover.jpg
coverCaption: "vim"

---

如何将一个vim打造成一个现代的IDE vim的使用有很多的基本操作
以及一些功能需要通过插件来实现，界面方面也要通过配置文件来实现想要的结果
这里主要的目的是记录使用配置`vim`的一些内容

<!-- more -->

## 基本使用

几种基本模式
```
    普通模式 ESC
    编辑模式 i o a
    视图模式 v
```
命令都是在普通模式下执行的
```
jkhl 上下左右
alt+jkhl 编辑模式移动光标

y v模式下copy选中的内容
yy 复制当前行
n yy 复制n行
x 剪切当前字符
d v模式下剪切选中的内容
dd 剪切
n dd 剪切n行
p 粘贴

gg 文档开头
G  文档末尾
n gg | :n 跳转到n行
u  undo 撤销修改
Ctrl+r redo

ctrl + f(front) 向下一页
ctrl + b(back)  向上一页

ctrl + d(down)  向下半页
ctrl + u(up)    向上半页

a 光标后追加
i 光标前插入
o 新建下一行
O 新建上一行
r 替换当前字符
^|0  行首
$    行尾

zz 将光标所在出移动到页面中间
H 定位到文件开头
L 定位到文件结尾
M 定位到文件中间

:w 保存
:q 退出
:q! 强制退出
:qa 关闭所有window
:e 重新载入文件内容 刷新文件内容
```
[参考链接](http://www.cnblogs.com/sunyubo/archive/2010/01/06/2282198.html)

## 基本配置
windows上面配置文件在vim运行目录下 `_vimrc`

基本设置
```
" 关闭与vi的兼容模式
set nocompatible
" 不生成~开头的备份文件
set nobackup 
" 不生成交换文件
set noswapfile
" 历史记录数 什么?
set history=1024
" 自动切换当前目录
set autochdir
" 去掉 utf-8 BOM
set nobomb
" Vim 的默认寄存器和系统剪贴板共享
set clipboard+=unnamed
" 设置 alt 键不映射到菜单栏
set winaltkeys=no
```

编码
```
" vim可识别的编码方式
set fileencodings=utf-8,gbk2312,gbk,gb18030,cp936
" 文件默认编码方式
set encoding=utf-8
" vim菜单栏语言
set langmenu=zh_CN
" vim默认语言
let $LANG = 'en_US.UTF-8'
```

界面

配色方案 [Tomorrow-Night](https://link.zhihu.com/?target=https%3A//github.com/chriskempson/tomorrow-theme/tree/master/vim/colors)
vim文件放在 `vim/vimfiles/colors` 这个文件夹

等宽字体下载 [Inconsolata](https://link.zhihu.com/?target=http%3A//www.fontex.org/download/inconsolata.otf)
字体安装不必详述

```
" 配色方案 
colorscheme Tomorrow-Night

source $VIMRUNTIME/delmenu.vim
source $VIMRUNTIME/menu.vim
" 高亮光标所在行
set cursorline
set hlsearch
" 显示行号
set number
" 窗口大小
set lines=35 columns=140
" 分割出来的窗口位于当前窗口下边/右边
set splitbelow
set splitright
"不显示工具/菜单栏
set guioptions-=T
set guioptions-=m
set guioptions-=L
set guioptions-=r
set guioptions-=b
" 使用内置 tab 样式而不是 gui
set guioptions-=e
set nolist
" 指定字体 以及字体 大小
set guifont=Inconsolata:h14:cANSI
```
格式设置
```
" 自动缩进
set autoindent
" 新行使用只能缩进
set smartindent
" 设定tab长度为4
set tabstop=4
" 空格代替缩进制表符
set expandtab
"
set softtabstop=4
" 
set foldmethod=indent
" 语法高亮
syntax on
```

## 插件
插件通过 `vundle` 来管理
在 `vim/vimfiles` 创建文件夹 `bundle`
通过git 下载插件 `git clone https://github.com/VundleVim/Vundle.vim.git`

需要的插件在 `vimrc` 进行配置
```
filetype off
set rtp+=$VIM/vinfiles/bundle/Vundle.vim/
call vundle#begin('$VIM/vimfiles/bundle/')

" 插件内容
Plugin 'VundleVim/Vundle.vim'

filetype on
call vundle#end()
```
然后在vim命令行模式安装 `:PluginInstall`


常用插件
NerdTree 文件目录结构树
```
Plugin 'scrooloose/nerdtree'
let NERDTreeIgnore=['.idea','.vscode','node_modules']
let NERDTreeBookmarksFile=$VIM . '/NerdTreeBookmarks'
let NERDTreeMinimalUI=1
let NERDTreeBookmarksSort=1
let NERDTreeShowLineNumbers=0
let NERDTreeShowBookmarks=1
let g:NERDTreeWinPos = 'right'
let g:NERDTreeDirArrowExpandable='→'
let g:NERDTreeDirArrowCollapsible='↓'
" 快捷键
nmap <leader>n :NERDTreeToggle<cr>

if exists('g:NERDTreeWinPos')
    autocmd vimenter * NERDTree D:\www
endif
```

Ctrlp 粘贴板共享
```
Plugin 'kien/ctrlp.vim'
let g:ctrlp_match_window='bottom,order,:btt,min:1,max:10,results:10'
set wildignore+=*\\.git\\*,*\\tmp\\*,*.swp,*.zip,*.exe
```

AirLine 下面的状态栏
```
Plugin 'vim-airline/vim-airline'
Plugin 'vim-airline/vim-airline-themes'
set laststatus=2

if !exists('g:airline_symbols')
    let g:airline_symbols = {}
endif
let g:airline_theme='tomorrow'
let g:airline_powerline_fonts = 1

let g:airline_symbols.branch = ''
let g:airline_left_sep = '»'
let g:airline_right_sep = '«'
let g:airline#extensions#branch#enabled = 1
let g:airline#extensions#branch#vcs_priority = ["git", "mercurial"]
let g:airline_mode_map = {
 \ 'n'  : 'N',
 \ 'i'  : 'I',
 \ 'v'  : 'V',
 \ }
let g:airline#extensions#tabline#enabled = 1
```

Neocomplete 代码提示 但是这个需vim有lua支持 具体如何支持看下面
```
Plugin 'Shougo/neocomplete.vim'
let g:acp_enableAtStartup=0
" Use neocomplete
let g:neocomplete#enable_at_startup=1
" Use smartcase
let g:neocomplete#enable_smart_case=1
let g:neocomplete#enable_auto_select=1
" Enable snipMate compatibility feature
let g:neosnippet#enable_snipmate_compatibility=1
" Tell Neosnippet about the other snippets
let g:neosnippet#snippets_directory=$VIM . '/vimfiles/bundle/vim-snippets/snippets'

" Enable omni completion.
autocmd FileType css setlocal omnifunc=csscomplete#CompleteCSS
autocmd FileType html,markdown setlocal omnifunc=htmlcomplete#CompleteTags
autocmd FileType javascript setlocal omnifunc=javascriptcomplete#CompleteJS
" php代码提示
autocmd FileType php setlocal omnifunc=phpcomplete#CompletePHP
autocmd Filetype xml setlocal omnifunc=xmlcomplete#CompleteTags
if !exists('g:neocomplete#sources#omni#input_patterns')
    let g:neocomplete#sources#omni#input_patterns = {}
endif
```


[参考地址](https://zhuanlan.zhihu.com/p/21328642)
配置文件参考 [配置文件地址](https://gist.github.com/keelii/1aab5f9aa5b47afa651c7fc84b8e9875)

## 编译vim使其支持lua
neocomplete是需要vim 有lua支持的
在windows上面想要vim支持lua需要你在windows 上重新编译vim
需要：
vim源代码 [下载](https://github.com/vim/vim)

lua解释器 [下载](http://tenet.dl.sourceforge.net/project/luabinaries/5.2.4/Tools%2520Executables/lua-5.2.4_Win64_bin.zip]以及 lua编译需要的文件 [下载](http://tenet.dl.sourceforge.net/project/luabinaries/5.2.4/Windows%2520Libraries/Dynamic/lua-5.2.4_Win64_dllw4_lib.zip)

编译环境 MinGW [下载地址](https://nuwen.net/files/mingw/mingw-14.0.exe)

将lua_bin 以及lua_lib 里面的文件放在一起


打开MinGW `open_distro_window.bat` cd 进入vin源码的src目录
执行make编译命令
```
make -f Make_ming.mak GUI=yes FEATURES=HUGE MBYTE=yes IME=yes GIME=yes DYNAMIC_IME=yes OLE=yes CSCOPE=yes DEBUG=no LUA="D:\Program Files (x86)\Vim\vim-lua\lua-vim" DYNAMIC_LUA=yes LUA_VER=52 USERNAME=fantasy USERDOMAIN=fantiq@163.com ARCH=x86-64 gvim.exe
```
注意里面的lua源码路径写你本地的lua路径

但是 我在`make` 编译的时候报错 
```
mkdir -p gobjx86-64 process_begin: CreateProcess(NULL, mkdir -p gobjx86-64, ...) failed. make (e=2): 系统找不到指定的文件。 make: *** [Make_cyg_ming.mak:860: gobjx86-64] Error 2 
```
这个问题是编译时候在执行 `mkdir -p gobjx86-64` 的时候 `MinGW` 并没有识别 `mkdir`的 `-p` 参数
这个问题可以通过手动创建 `gobjx86-64` 文件夹之后再 执行 make 来解决


编译成功后会在当前文件夹生成 `Gvim.exe` 文件
将这个文件以及 `lua52.dll` 复制到 vim 的runtime目录`vim/vim74` Gvim.exe 进行覆盖


[参考链接](https://zhuanlan.zhihu.com/p/21328642)

