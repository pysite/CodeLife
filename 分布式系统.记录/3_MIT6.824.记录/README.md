> 学习MIT6.824的记录

首先推荐一个别人写的文档：https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/

再推荐另一个别人写的文档：https://zhenghe.gitbook.io/open-courses/

再推荐B站上的带字幕的视频：https://www.bilibili.com/video/BV1R7411t71W?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click&vd_source=20a0ef163ccfc72025d3725383a16564

MIT课程主页：https://pdos.csail.mit.edu/6.824/schedule.html

MIT推荐的go语言教程：https://go.dev/tour/welcome/1，中文版：https://tour.go-zh.org/welcome/1



### 虚拟机ubuntu1804连不上网

sudo vim 00-installer-config.yaml

将里面的dns server地址改为114.114.114.114后恢复正常

### 虚拟机ubuntu1804软件源更换为华为云源

在source.list必须为初始的情况下照着https://mirrors.huaweicloud.com/home里面说的做就行

实测在公司里速度可达10MB/s

### vscode提示the gopls command不可用

```bash
$ go env -w GO111MODULE=on
$ go env -w GOPROXY=https://goproxy.io,direct
```

再在vscode中点击"install all"

### go RPC入门

https://blog.csdn.net/weixin_45413603/article/details/121514386

Go runs the handler for each RPC in its own thread, so the fact that one handler is waiting won't prevent the coordinator from processing other RPCs.