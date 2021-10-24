进入/退出：``tmux`` / ``exit``



第一个启动的 Tmux 窗口，编号是`0`，第二个窗口的编号是`1`，以此类推。



会话是指执行的进程们，而窗口则是我们看到的，一般来说一个会话绑定一个窗口，当窗口关闭其对应的会话中的进程也被终止，而tmux detach就可以解除这种绑定，而且tmux还可以一个会话绑定到多个窗口，因此可以实现实时共享。



以一个名称创建新会话，默认是数字ID些：

```bash
$ tmux new -s <session-name>
```



分离会话和窗口：

```bash
$ tmux detach
```



查看当前有哪些会话：

```bash
$ tmux ls
# 等价于
$ tmux list-session
```



接入正在运行的会话：

```bash
# 使用会话编号
$ tmux attach -t 0

# 使用会话名称
$ tmux attach -t <session-name>
```



终止某个会话：

```bash
# 使用会话编号
$ tmux kill-session -t 0

# 使用会话名称
$ tmux kill-session -t <session-name>
```



切换会话：

```bash
# 使用会话编号
$ tmux switch -t 0

# 使用会话名称
$ tmux switch -t <session-name>
```



重命名会话：

```bash
$ tmux rename-session -t 0 <new-name>
```



会话快捷键：

- `Ctrl+b d`：分离当前会话。
- `Ctrl+b s`：列出所有会话。
- `Ctrl+b $`：重命名当前会话。



划分窗格：

```bash
# 划分上下两个窗格
$ tmux split-window

# 划分左右两个窗格
$ tmux split-window -h
```



移动光标：

```bash
# 光标切换到上方窗格
$ tmux select-pane -U

# 光标切换到下方窗格
$ tmux select-pane -D

# 光标切换到左边窗格
$ tmux select-pane -L

# 光标切换到右边窗格
$ tmux select-pane -R
```



交换窗格位置：

```bash
# 当前窗格上移
$ tmux swap-pane -U

# 当前窗格下移
$ tmux swap-pane -D
```



窗格快捷键：

- `Ctrl+b %`：划分左右两个窗格。
- `Ctrl+b "`：划分上下两个窗格。
- `Ctrl+b <arrow key>`：光标切换到其他窗格。`<arrow key>`是指向要切换到的窗格的方向键，比如切换到下方窗格，就按方向键`↓`。
- `Ctrl+b ;`：光标切换到上一个窗格。
- `Ctrl+b o`：光标切换到下一个窗格。
- `Ctrl+b {`：当前窗格与上一个窗格交换位置。
- `Ctrl+b }`：当前窗格与下一个窗格交换位置。
- `Ctrl+b Ctrl+o`：所有窗格向前移动一个位置，第一个窗格变成最后一个窗格。
- `Ctrl+b Alt+o`：所有窗格向后移动一个位置，最后一个窗格变成第一个窗格。
- `Ctrl+b x`：关闭当前窗格。
- `Ctrl+b !`：将当前窗格拆分为一个独立窗口。
- `Ctrl+b z`：当前窗格全屏显示，再使用一次会变回原来大小。
- `Ctrl+b Ctrl+<arrow key>`：按箭头方向调整窗格大小。
- `Ctrl+b q`：显示窗格编号。



新建窗口：（这个新建的窗口不会并排显示，注意**窗口**window和**窗格**pane的区别）

```bash
$ tmux new-window

# 新建一个指定名称的窗口
$ tmux new-window -n <window-name>
```



切换窗口：

```bash
# 切换到指定编号的窗口
$ tmux select-window -t <window-number>

# 切换到指定名称的窗口
$ tmux select-window -t <window-name>
```



重命名当前窗口：

```bash
$ tmux rename-window <new-name>
```



窗口快捷键：

- `Ctrl+b c`：创建一个新窗口，状态栏会显示多个窗口的信息。
- `Ctrl+b p`：切换到上一个窗口（按照状态栏上的顺序）。
- `Ctrl+b n`：切换到下一个窗口。
- `Ctrl+b <number>`：切换到指定编号的窗口，其中的`<number>`是状态栏上的窗口编号。
- `Ctrl+b w`：从列表中选择窗口。
- `Ctrl+b ,`：窗口重命名。



其他命令：

```bash
# 列出所有快捷键，及其对应的 Tmux 命令
$ tmux list-keys

# 列出所有 Tmux 命令及其参数
$ tmux list-commands

# 列出当前所有 Tmux 会话的信息
$ tmux info

# 重新加载当前的 Tmux 配置
$ tmux source-file ~/.tmux.conf
```