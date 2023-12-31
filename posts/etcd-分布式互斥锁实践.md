---
title: etcd-分布式互斥锁实践
description: etcd-分布式互斥锁实践
date: 2023-12-31
tags:
  - etcd

---

在单进程中，我们可以使用标准库中的互斥锁sync.Mutex来实现资源的并发访问。但在分布式系统中，各个进程运行在不同的节点，sync.Mutex已无法满足要求。此处，我们来体验一下etcd提供的分布式互斥锁。

### 1. 互斥测试

测试代码如下，启动三个协程模拟三个不同的客户端，每个协程都会创建一个新的客户端去争抢互斥锁



```go
package main

import (
	clientv3 "go.etcd.io/etcd/client/v3"
	"go.etcd.io/etcd/client/v3/concurrency"
	"log"
	"sync"
	"time"
)

var (
	endpoints = []string{"http://192.168.92.132:2379"}
	lockName  = "test-lock" // 锁前缀
)

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(j int) {
			defer wg.Done()
			f(j)
		}(i)
	}
	wg.Wait()
}

func f(n int) {
	// 连接etcd
	cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	// 获取session
	session, err := concurrency.NewSession(cli, concurrency.WithTTL(10))
	if err != nil {
		log.Fatal(err)
	}
	defer session.Close()

	// 创建锁
	m := concurrency.NewLocker(session, lockName)
	m.Lock()
	log.Printf("客户端 %v 获取锁\n", n)

	time.Sleep(time.Second * 3)

	m.Unlock()
	log.Printf("客户端 %v 释放锁\n", n)
}
```



运行结果如下，三个客户端已实现互斥

```
2023/12/31 15:16:28 客户端 2 获取锁
2023/12/31 15:16:31 客户端 2 释放锁
2023/12/31 15:16:31 客户端 0 获取锁
2023/12/31 15:16:34 客户端 0 释放锁
2023/12/31 15:16:34 客户端 1 获取锁
2023/12/31 15:16:37 客户端 1 释放锁
```

### 2. 死锁测试

假设其中一个客户端获取锁后崩溃了，锁没有释放，会造成死锁吗？我们来测试一下，修改一下代码，当客户端2获取锁后直接return模拟宕机，看看有什么效果。代码如下：

```go
func f(n int) {
	// 连接etcd
	cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	// 获取session
	session, err := concurrency.NewSession(cli, concurrency.WithTTL(10))
	if err != nil {
		log.Fatal(err)
	}
	defer session.Close()

	// 创建锁
	m := concurrency.NewLocker(session, lockName)
	m.Lock()
	log.Printf("客户端 %v 获取锁\n", n)

	if n == 2 {
		return
	} else {
		time.Sleep(time.Second * 3)
	}

	m.Unlock()
	log.Printf("客户端 %v 释放锁\n", n)
}
```



运行结果如下，假设某个客户端崩溃或忘记释放锁，并不会造成死锁

```
2023/12/31 15:28:49 客户端 1 获取锁
2023/12/31 15:28:52 客户端 1 释放锁
2023/12/31 15:28:53 客户端 2 获取锁
2023/12/31 15:28:53 客户端 0 获取锁
2023/12/31 15:28:56 客户端 0 释放锁
```



# 3. 参考资料

- https://etcd.io/docs/v3.5/dev-guide/api_concurrency_reference_v3/
- https://juejin.cn/post/7062900835038003208

