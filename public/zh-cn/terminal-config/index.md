# 终端配置总结


## 终端配置总结

### 背景

最近公司新购置了好几台 Linux 服务器然后配置一些服务的时候很不习惯，估计是我平时自己的 zsh + oh-my-zsh 用多了，故想整理下 .bash_rc 和 .zshrc 我个人的一些配置，这些配置包含了一些 alias 快捷命令和命令行系统配置，可以让终端变得快捷易用，今晚再写个 shell 脚本实现快速修改 .bash_rc 的配置。
<!--more-->
### .bash_rc/.zshrc 配置汇总

- 终端不自动执行命令

  ``` bash
  # If not running interactively, don't do anything
  case $- in
      *i*) ;;
        *) return;;
  esac
  ```

- .bash_history 文件(同理 .zsh_history )不重写而是使用附加模式记录命令

  ``` bash
  # append to the history file, don't overwrite it
  shopt -s histappend
  ```

- 增加 .bash_history 、 .zsh_history 文件记录阈值，超过阈值后自动清空

  ``` bash
  HISTFILESIZE=2000
  ```

- 根据命令行长短自动调节终端行列显示排版

  ``` bash
  # check the window size after each command and, if necessary,
  # update the values of LINES and COLUMNS.
  shopt -s checkwinsize
  ```

- 开启适合编程的命令行提示 feature

  ``` bash
  if ! shopt -oq posix; then
    if [ -f /usr/share/bash-completion/bash_completion ]; then
      . /usr/share/bash-completion/bash_completion
    elif [ -f /etc/bash_completion ]; then
      . /etc/bash_completion
    fi
  fi
  ```

- 常用的 alias 声明 — 简写部分

  比较容易懂。

  ``` bash
  alias cls='clear'
  alias vi='vim'
  alias ll='ls -l'
  alias la='ls -a'
  ```

- 常用的 alias 声明 — 效果增强部分

  这部分让 ls 和 grep 都带有关键字亮色的提示，让 alert 提示的错误信息在终端中显示起来更友好。

  ``` bash
  alias ls='ls --color=auto'
  alias grep='grep --color=auto'
  alias alert='notify-send --urgency=low -i "$([ $? = 0 ] && echo terminal || echo error)" "$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert$//'\'')"'
  ```

- 常用的 alias 声明 — 效果增强部分2

  这部分增加了在 MacOS 系统中显示和隐藏文件的快捷命令，和在终端中切换 bash 和 zsh 的快捷命令。

  ``` bash
  alias showfile='defaults write com.apple.finder AppleShowAllFiles -boolean true ; killall Finde    r'
  alias hidefile='defaults write com.apple.finder AppleShowAllFiles -boolean false ; killall Find    er'
  alias switchbash='chsh -s /bin/bash'
  alias switchzsh='chsh -s /bin/zsh'
  ```

全部的配置汇总如上，一是方便我自己做个备份，二是大家可以按需使用。

