---
layout: post
title:  "insert time in vim"
date:   2015-08-12 20:32:47
categories: jekyll update
---

在vim中快速插入当前系统时间，可以通过修改vim的配置文件，将F5映射到打印系统时间。

这就涉及到修改vim的配置文件。vimrc是vim最主要的配置文件，它有两个版本，全局版本（global）和用户版本（personal）。全局vimrc文件在Vim的安装目录中，我的电脑是Mac，所以其路径是
/usr/share/vim/vimrc，用户版本的vimrc文件在当前用户的主目录下，主目录的位置依赖于操作系统。Mac下的用户vimrc文件路径为：
/Users/用户名/.vimrc。但是Mac下默认是没有用户vimrc的，所以需要自己创建一个。用户版的vimrc会覆盖全局版。

    :nnoremap <F5> "=strftime("%F")<CR>gP
    :inoremap <F5> <C-R>=strftime("%F")<CR>

将上面两行加入$HOME/.vimrc中，就可以在一般模式和编辑模式下用快捷键F5，插入当前系统时间了。这里设置的时间格式是xxxx-xx-xx。可以按照需要修改为自己想要的模式。比如修改为`YYYY-MM-DD HH:MM:SS`，可以将%F替换为`\%Y-\%m-\%d \%H:\%M:\%S`。

至此，大功告成。
