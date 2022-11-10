> MIT 6.824 Spring 2020
>
> 发布到知乎，用于督促自己学习，笔记中可能有错误，还望各位见谅。

Go语言在多线程、锁、线程间同步上非常便利，Go的RPC包也相当的方便。

Go是类型安全（type safe）和内存安全（memory safe）的语言，Go的垃圾回收机制能尽可能帮助消灭大量相关bug，例如double-free。

> 参考阅读《Effective Go》
> Goroutine便是Go语言的thread。

每个线程拥有自己的程序计数器（PC）、一套寄存器、一个栈，一个进程的多个线程的栈共享一个地址空间。



使用多线程的理由：

1. I/O concurrency
   多线程是实现IO并发的理想方法，例如可以多个线程同时发起多个RPC调用并等待回应。
2. Parallelism并行化
   多线程能够充分发挥多核CPU的算力。

3. Convenience
   一些后台工作、周期性执行的工作可以交给其它线程来完成，而不需要主函数一直关心。



如果不使用多线程，而使用事件驱动编程（这两种方式也可以结合起来使用），事件驱动模式通常有一个主循环，这个循环等待输入或者其他事件来执行相关代码。这种方式的缺点就是可能没办法很好地利用CPU的并行化（同时利用多个核）；多线程方式的缺点就是当线程数量很多时，会因为每个线程分配个栈而占用大量内存，而且线程间的调度也会变得很耗时。



> goroutine是用户态线程，不受OS调度，OS每次选择执行go时，go再在自己的goroutines中选择一个来执行。go会进行自己的routine切换。Goroutine调度器（调度器处在Go运行时中）按照一定算法将goroutines放到不同的操作系统线程中去执行。



使用多线程的挑战：

1. Shared data共享数据
   经典"n=n+1"竞争（race）问题。
2. Coordination
   例如生产者消费者队列。
   go channel、condition variables、wait group。

2. Deadlock



> 下面是课程中展示的使用go写的爬虫代码；
>
> go语言中map默认就是个指针，所以``func Serial``参数中不需要加``*``；
> 通过``defer``保证函数即使fail也能在退出时执行某个逻辑；
> go会把内部定义的函数使用到的外部变量给存放到堆里，这样即使外部函数退出了，内部函数的goroutine依然可以使用；（go的变量逃逸分析）

```go
package main

import (
	"fmt"
	"sync"
)

//
// Several solutions to the crawler exercise from the Go tutorial
// https://tour.golang.org/concurrency/10
//

//
// Serial crawler
//

func Serial(url string, fetcher Fetcher, fetched map[string]bool) {
	if fetched[url] {
		return
	}
	fetched[url] = true
	urls, err := fetcher.Fetch(url)
	if err != nil {
		return
	}
	for _, u := range urls {
		Serial(u, fetcher, fetched)
	}
	return
}

//
// Concurrent crawler with shared state and Mutex
//

type fetchState struct {
	mu      sync.Mutex
	fetched map[string]bool
}

func ConcurrentMutex(url string, fetcher Fetcher, f *fetchState) {
	f.mu.Lock()
	already := f.fetched[url]
	f.fetched[url] = true
	f.mu.Unlock()

	if already {
		return
	}

	urls, err := fetcher.Fetch(url)
	if err != nil {
		return
	}
	var done sync.WaitGroup
	for _, u := range urls {
		done.Add(1)
        // 不这么做的话, 内部函数真正执行时看到的u的值可能已经改变, 因为内部函数的启动是异步的
        u2 := u	
		go func() {
			defer done.Done()
			ConcurrentMutex(u2, fetcher, f)
		}()
		//go func(u string) {
		//	defer done.Done()
		//	ConcurrentMutex(u, fetcher, f)
		//}(u)
	}
	done.Wait()
	return
}

func makeState() *fetchState {
	f := &fetchState{}
	f.fetched = make(map[string]bool)
	return f
}

//
// Concurrent crawler with channels
//

func worker(url string, ch chan []string, fetcher Fetcher) {
	urls, err := fetcher.Fetch(url)
	if err != nil {
		ch <- []string{}
	} else {
		ch <- urls
	}
}

func master(ch chan []string, fetcher Fetcher) {
	n := 1	// n是chanel中的消息数量
	fetched := make(map[string]bool)
	for urls := range ch {
		for _, u := range urls {
			if fetched[u] == false {
				fetched[u] = true
				n += 1
				go worker(u, ch, fetcher)
			}
		}
		n -= 1
		if n == 0 {
			break
		}
	}
}

func ConcurrentChannel(url string, fetcher Fetcher) {
	ch := make(chan []string)
    // 这里必须另起一个goroutine, 因为是无缓冲channel, 会一直等待直到消息被读取
	go func() {
		ch <- []string{url}
	}()
	master(ch, fetcher)
}

//
// main
//

func main() {
	fmt.Printf("=== Serial===\n")
	Serial("http://golang.org/", fetcher, make(map[string]bool))

	fmt.Printf("=== ConcurrentMutex ===\n")
	ConcurrentMutex("http://golang.org/", fetcher, makeState())

	fmt.Printf("=== ConcurrentChannel ===\n")
	ConcurrentChannel("http://golang.org/", fetcher)
}

//
// Fetcher
//

type Fetcher interface {
	// Fetch returns a slice of URLs found on the page.
	Fetch(url string) (urls []string, err error)
}

// fakeFetcher is Fetcher that returns canned results.
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) ([]string, error) {
	if res, ok := f[url]; ok {
		fmt.Printf("found:   %s\n", url)
		return res.urls, nil
	}
	fmt.Printf("missing: %s\n", url)
	return nil, fmt.Errorf("not found: %s", url)
}

// fetcher is a populated fakeFetcher.
var fetcher = fakeFetcher{
	"http://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"http://golang.org/pkg/",
			"http://golang.org/cmd/",
		},
	},
	"http://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"http://golang.org/",
			"http://golang.org/cmd/",
			"http://golang.org/pkg/fmt/",
			"http://golang.org/pkg/os/",
		},
	},
	"http://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
	"http://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
}
```

