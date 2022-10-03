> 教学视频 https://www.youtube.com/watch?v=PorfLSr3DDI
>
> https://undo.io/resources/gdb-watchpoint/5-ways-reduce-debugging-hours/

gcc -g hello.c 其中-g是指编译带调试信息

gdb hello.out

(gdb)start 等于在main处打断点并且run

(gdb)list 打印当前所处程序的位置的前后4行代码

(gdb)next 执行下一句（遇到函数不进入直接执行），等同于n

(gdb)step 执行下一句 （遇到函数进入），等同于s

(gdb)b main 在函数main打断点

(gdb)b 9 在文件第9行打断点

(按住ctrl) x a 进入TUI mode

(按住ctrl) L 刷新屏幕

(按住ctrl) x 2 调整当前TUI的界面风格（是否展示汇编代码等）

(按住ctrl) x 1 回到command window

在TUI mode中不能按上下左右方向键来切换command，必须 ctrl-p 或 ctrl-n 或 ctrl-b 或 ctrl-f

(gdb)tui reg float 以浮点数形式展示寄存器内容



在gdb7之后，内嵌python

(gdb)python print('hello world')

(gdb)python print(gdb.breakpoints()[0].location)

(gdb)python gdb.Breakpoint('7') 用python来断点



如果程序错误且生成了core dump则可以

gdb -c core.14276 这里的core.14276是生成的core dump文件

(gdb)bt 打印调用栈



(gdb) b _exit.c:32 在exit.c文件的32行打断点

```c
// 当触发断点3时执行gdb run指令
(gdb) command 3
    run
    end
// 当触发断点2时执行record和continue指令
(gdb) command 2
    record	// 让程序开始记录反向调试所必要的信息，其中包括保存程序每一步运行的结果等等信息，所以如果没有运行此指令，是没有办法进行反向调试的，停止反向调试使用的是record stop ，这样反向调试的记录停止，可以正常运行程序了。
    continue // TUI中continue命令通知调试器恢复执行并继续,直到遇到断点为止
    end
```

(gdb)set pagination off 不让gdb分页显示



如果gdb运行过程中遇到程序出错

(gdb)p $pc 打印寄存器信息

(gdb) x 地址 显示内存地址中的数据，比如显示pc寄存器中所指地址的数据

(gdb)bt

(gdb)reverse-stepi 回到上一条指令执行前名（**巨有用**，当bt显示不出信息时记得先返回一条指令再bt）

(gdb)disas 反汇编，即显示当前所处的机器指令位置

(gdb)print $sp 打印sp寄存器的值

(gdb)print *(long **) 0x7fffffffdc98 打印内存0x7fffffffdc98存储的值

(gdb)watch *(long **) 0x7fffffffdc98 如果后面这个表达式发生改变则程序停止运行

(gdb)reverse-continue 一直回退直到gdb中断

(gdb)print i 打印程序中名为i的变量的值