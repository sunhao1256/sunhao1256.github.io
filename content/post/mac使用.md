---
title: "mac使用"
date: 2022-06-20T16:57:01+08:00
tags: ['mac']
---



# 快捷键

- Finder绝对路径跳转

  command+shift+G

- Finder绝对路径复制

  command+option+C

- 表情

  control+cmmand+space

- 录屏

  Shift-Command-5

- 截图

  command shift 4局部，3全屏

- 只显示桌面

  command f3
  
- finder

  option+command+空格
  
- space+command 

  打开聚焦搜索，按住command再点击结果，是进入文件夹
  
- from current terminal open finder

  tap open <path>

# 环境变量

加载顺序

```
/etc/profile
/etc/paths 
~/.zprofile 
~/.zshrc
```

/etc/profile和/etc/paths是系统级别的，系统启动就会加载，zprofile是用户级别的
~/.zshrc没有上述规则，它是zsh shell打开的时候载入的。

```shell
# System-wide .profile for sh(1)

if [ -x /usr/libexec/path_helper ]; then
        eval `/usr/libexec/path_helper -s`
fi

if [ "${BASH-no}" != "no" ]; then
        [ -r /etc/bashrc ] && . /etc/bashrc
fi
```

**path_helper**是将/etc/paths和/etc/paths.d中定义的路径,加入环境变量. 苹果推荐使用这个

Mac下的Java,默认会直接装好Java命令.是通过**/usr/libexec/java_home**命令找环境变量中的JAVA_HOME变量,用于输出Java

# Iterm2

command快捷键

- ctrl+a

  行首

- ctrl+e

  行尾

- option+b

  前一个单词

- option+f

  后一个单词

# Chrome

ctrl+shift+n打开一个无痕

# Karabiner 

改键神器