---
layout: post
title: tmux 打造高效开发环境
tags:
  - productivity
  - tmux
  - terminal
  - linux
date: 2019-11-15
---

开发环境里，要启动各种服务，操作 `git`，调试，有时还要管理远程服务器。往往一天工作下来，可能需要打开8-10个 `terminal`。要想在茫茫窗口里找到想要的那个，是个考验眼力的活。万一一眼看错选了别的，重新再来一遍也是个很烦人的过程。而一个好用的窗口管理工具，会让你游刃有余地在多个窗口间切换，如臂使指，从容面对纷繁复杂的工作任务。

要介绍终端窗口管理工具，就绕不开一个概念，[终端复用（terminal multiplexer）](https://en.wikipedia.org/wiki/Terminal_multiplexer)。所谓终端复用，是指在一个固定的窗口内，集中管理多个不同的会话，可以是本地 `shell`，也可以是编辑器，例如 `vim`，或者是远程服务器 `SSH` 连接。而终端复用就是解决上面说到的多窗口管理问题的利器，同时，用在服务器上的话，还可以解决远程执行长时间命令客户端断开导致操作终止的尴尬。

`Linux` 上的终端复用工具，古早的有 [GNU screen](https://www.gnu.org/software/screen/)，1987年发布，自由软件项目的一员，具备完善的多 `console` 管理功能，支持会话 `attach` 和 `detach`，而且大多数发行版的默认软件源都提供安装包，适合用在远程服务器的管理。

[tmux](https://github.com/tmux/tmux/wiki) 是2007年发布的现代化工具，具备 `screen` 的全部功能，并且支持可编程脚本。而也就是可编程能力，让 `tmux` 一骑绝尘，远超古早 `GNU screen` 的可用性。

安装好完成后，终端输入 `tmux` 即可使用

![Default tmux UI](/assets/images/4ae5b1aa61faa5b987419b48750de7e4/default-tmux.png)

如果你没注意到的话，这个默认的 `tmux` 界面有一个状态栏，左侧是窗口列表，使用者可以通过它知道自己当前在哪个会话中。右侧是一些状态信息，默认提供了主机名、时间和日期。这些信息现在看起来还非常简陋，似乎没有多大帮助。别担心，这些都可以定制，后面会说到这部分内容。

一般来说，终端复用工具需要一个前导键，以区分工具本身的操作指令和一般键盘输入，`tmux` 的默认前导键是 `Ctrl-b`。例如，启动 `tmux` 后希望创建另一个窗口，在 `tmux` 里这个操作可以用命令 `c` 完成，那么，实际的键盘输入应该是 `Ctrl-b + c` ，这样一个新的序号为1的窗口就出现了。

![Extra 1 more window in tmux](/assets/images/4ae5b1aa61faa5b987419b48750de7e4/tmux-1-more-window.png)

在状态栏里可以看到这个新的窗口。这时可以使用前导键+窗口序号数字切换到指定的窗口，例如 `Ctrl-b+0` 就可以离开新建窗口回到之前的默认窗口。

如果你是 `vim` 用户，看到这里可能会嘀咕，`Ctrl-b` 用来做前导键了，那 `vim` 里怎么翻页呢？我们先来解决这个问题。

在 `tmux` 启动时，会首先加载系统配置文件 `/etc/tmux.conf`，随后加载用户配置 `~/.tmux.conf`。而所谓的可编程脚本，就在这些配置文件中。

先来看看怎么解决前导键冲突问题，在 `~/.tmux.conf` 加入下面一行（如果没有，就创建一个新文件）：

```tmux
set-option -g prefix C-a
```

`set-option` 是 `tmux` 命令，`-g` 表示全局设置，`prefix C-a` 将前导键设为 `Ctrl-a`，从而释放 `vim` 的常用组合键 `Ctrl-b`。

另一个问题，从前面的图2可以看到，`tmux` 中的窗口序号从 0 开始。看看键盘，`Ctrl-a`前导键在键盘左侧，而数字0则远在键盘右侧，需要双手协作才能完成切换动作，不是最优的移动策略。没关系，在 `~/.tmux.conf` 中增加一行：

```tmux
set -g base-index 1
```

这里的 `set` 是上面一行 `set-option` 的别名。`tmux` 很多命令都有别名，例如 `attach-session` 可以简写为 `attach`，`new-window` 简写成 `neww` 等等。利用别名可以在自定义配置的时候可以少敲一些键盘。

`tmux` 在创建新窗口时，会从一个基准开始，找到最小的当前未被使用的值即为新窗口的序号。`base-index 1` 是告诉 `tmux` 查找的基准值从1开始，不再是默认的0。这样，0号窗口再也不会出现在当前 `session` 里。

这里又出现一个新的概念，`session`。简而言之，`tmux` 启动时创建的是一个 `session`，而一个 `session` 内可以包含多个 `window`（窗口），每个 `window` 又可以被切分为多个 `pane`（`%` 左右切分，`"` 上下切分）。

![Session, Window and Panes](/assets/images/4ae5b1aa61faa5b987419b48750de7e4/tmux-session-window-pane.png)

用户配置文件还可以进一步丰富：

```tmux
set -g status-interval 1        # 设置状态栏刷新间隔1秒
set -g status-justify centre    # 居中窗口列表
set -g status-left-length 20    # 设置左侧状态栏最大长度
set -g status-right-length 140  # 设置右侧状态栏最大长度
set -g status-left '#[fg=green]#H #[fg=black]• #[fg=green,bright]#(uname -r | cut -c 1-6)#[default]' # 设置左侧显示主机名，内核版本
set -g status-right '#[fg=red,dim,bg=default]#(uptime | cut -f 4-5 -d " " | cut -f 1 -d ",") #[fg=white,bg=default]%a %l:%M:%S %p#[default] #[fg=blue]%Y-%m-%d' # 设置右侧显示 uptime, 时间和日期

# 使用 vim 风格的 pane 跳转按键
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# 使用前导键+r重新加载用户配置
bind r source-file ~/.tmux.conf \; display-message "Config reloaded..."
```

这样，一个基本可用的 `tmux` 初步完工。

`tmux` 的自动化能力如此强大，如果仅仅用来做一些花花绿绿的装饰工作，未免大材小用。

假设日常工作中要用到一个 `git` 操作终端，一个 `docker-compose` 启动的一组容器，以及后台数据服务。同时，远程服务器上还有一些维护工作。手动启动所有这些终端、服务、命令行，作为每日常规操作，枯燥繁琐。`tmux` 的自动化脚本可以一键完成所有的日常操作，你只需要全身心投入更具创造性的工作即可。

```tmux
# tmux-work-env.conf
#
# 首先加载默认用户配置，这里是一些普适选项，修改前导键，设置窗口起始序号等
source-file ~/.tmux.conf

# 创建工作区会话
new-session -d
# 本会话的默认窗口作为 git 操作终端，进入目录，并自动 git status
send-keys 'cd /path/to/git-repo' Enter
send-keys 'git status' Enter

# 横向切分当前窗口，并设置工作目录到 docker-compose.yml 目录
split-window -h -c /path/to/docker-compose.yml
send-keys 'docker-compose up' Enter
# 焦点重新回到 git 终端
select-pane -t 0

# 创建后台服务新窗口，同时设置工作目录
new-window -c /path/to/backend/service
# make 并且启动后台服务
send-keys 'make && ./REALLY_INTERESTING_SERVICE' Enter

# 创建 SSH 会话窗口并启动 SSH 连接
new-window
send-keys 'SSH xxx@192.168.1.10' Enter

# 创建备用 shell，工作目录 $HOME
new-window -c ~
```

保存为文件 `tmux-work-env.conf` 后，在命令行使用

```bash
tmux -f tmux-work-env.conf attach
```

启动你的自动化工作区吧。

`tmux` 自动化的使用场景非常丰富，网上也可以找到各种各样的案例，可以对照自己的实际工作流程按需取用。同时，`man tmux` 是我遇到问题的第一解决手段，`tmux` 文档详细丰富，常见问题基本都能找到答案。
