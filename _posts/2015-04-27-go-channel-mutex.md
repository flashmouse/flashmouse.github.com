---
layout: post
title: 一次失败的channel-map实验
categories: [golang]
tags: [golang]
---

刚刚入门golang，尝试写一个push系统，需要一个“线程安全”的map，看了看golang的库，发现没有提供，官网说法是考虑再三没有实现。然后我在github上发现一份基于rwmutex的[并发map源码](https://github.com/streamrail/concurrent-map)。这个时候我一脑抽，觉得用了golang就要用channel！然后决定用channel实现一个。不过结局并不好，证明shared memory的通信方式还是用mutex更简便，更有效率。

我觉得非常有搞头的在github上创建了新的[项目](https://github.com/flashmouse/go-channel-map/)。代码不多也贴上凑字数：

```go
package chmap

import "hash/fnv"

//impl Imap
type Chmap struct {
	innerMaps []innerMap
	shardNum  int
}

type innerMap struct {
	kv map[string]interface{}
	request chan mission
}

var (
	PUT int = 1
	DEL int = 2
	GET int = 3
)

type kv struct {
	k string
	v interface{}
}

type mission struct{
	mission_type int
	kv           kv
}

func NewMap(shardNum int) *Chmap {
	v := Chmap{}
	v.shardNum = shardNum
	v.init()
	return &v
}

func (v *Chmap) init() {
	v.innerMaps = make([]innerMap, v.shardNum, v.shardNum)
	for i := 0; i < v.shardNum; i++ {
		v.innerMaps[i] = innerMap{make(map[string]interface{}), make(chan mission)}
		go v.innerMaps[i].init()
	}
}

func (m *Chmap) getShard(k string) uint {
	h := fnv.New32()
	h.Write([]byte(k))
	return uint(h.Sum32()) % uint(m.shardNum)
}

func (m *Chmap) Delete(k string) {
	m.innerMaps[m.getShard(k)].request <- mission { DEL, kv{k, nil} }
}

func (m *innerMap) init() {
	for mission := range m.request {
		switch mission.mission_type {
		case DEL:
			delete(m.kv, mission.kv.k)
		case PUT:
			m.kv[mission.kv.k] = mission.kv.v
		}
	}
}

func (m *Chmap) Get(k string) (interface{}, bool) {
	v , ok := m.innerMaps[m.getShard(k)].kv[k]
	return v, ok
}

func (m *Chmap) Put(k string, v interface{}) {
	m.innerMaps[m.getShard(k)].request <- mission {PUT, kv{k, v}}
}

func (m *Chmap) Count() int {
	re := 0
	for i := 0; i < m.shardNum; i++ {
		re += len(m.innerMaps[i].kv)
	}
	return re
}

func (m *Chmap) Contain(k string) bool {
	_, ok := m.innerMaps[m.getShard(k)].kv[k]
	return ok
}
```

虽然没有注释不是好习惯但我已经很难纠正……

总体思路和网上那份rwmutex的实现很像，应该也是和jdk的ConcurrenHashMap实现思路相近。一个map里实际上了分了的个段，造成了一定的资源浪费的同时降低并行竞争的概率以提高并发速度。

我还没有对读操作进行规划，不过这不要紧，因为这个时候我已经开始尝试测试性能了……

我写了以下代码测试我的map的速度：

```go
package main

import (
	"github.com/flashmouse/go-channel-map/chmap"
	"math/rand"
	"time"
	"fmt"
	"runtime"
	"strconv"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(runtime.NumCPU())
	shardNum, maxKey, threadNum , loop := 32, 128, 10, 10000000 / 2
	cmap := chmap.NewMap(shardNum)

	okFlag := make(chan int)

	beginTime, endTime := time.Now(), time.Now()

	for i := 0; i < threadNum; i++ {
		go func() {
			r := rand.New(rand.NewSource(time.Now().Unix()))
			for j := 0; j < loop; j++ {
				key := r.Intn(maxKey)
				cmap.Put(strconv.Itoa(key), &key)
			}

			okFlag <- 1
		}()
	}

	exitFlag := sync.WaitGroup{}
	exitFlag.Add(1)

	go func() {
		flag := 0
		for v := range okFlag {
			if v == 0 {
				break
			}
			flag ++
			fmt.Println("get a ok flag")
			if (flag >= threadNum) {
				break
			}
		}
		endTime = time.Now()
		fmt.Printf("use time: %d", (endTime.Unix()-beginTime.Unix()))
		close(okFlag)
		exitFlag.Done()
	}()

	exitFlag.Wait()
}
```

然后又写了同样的一份测试concurrent-map。

很不幸，我的channel-map耗时17秒，concurrent-map耗时仅为10秒。如此差距令人心醉……

难道channel比mutex性能低这么多？于是我又写了个测试……

```go
package main

import (
	"sync"
	"time"
	"fmt"
)

func main() {
	loop := 100000 * 1000 / 4

	lock := sync.Mutex{}

	maps := make(map[int]int)

	beginTime := time.Now()
	for i := 0; i < loop; i++ {
		lock.Lock()
		if (i < 0) {
			break
		}
		maps[1] = 1
		lock.Unlock()
	}
	fmt.Println(time.Since(beginTime))

	c := make(chan int)

	go func() {
		for v := range c {
			if (v < 0) {
				break
			}
			maps[1] = 1
		}
	}()

	beginTime = time.Now()
	for i := 0; i < loop; i++ {
		c<-i
	}
	fmt.Println(time.Since(beginTime))
}

type hehe struct{
	a int
	b string
}
```

执行结果为：
```
/usr/local/go/bin/go run /Users/lxy/Documents/go/src/github.com/flashmouse/go-channel-map/test/test.go
1.269987278s
5.678916621s

Process finished with exit code 0
```

看来channel在性能上有所牺牲毋容置疑。不过除了上面这个测试，我还做了两个测试：

* 在我自己的机器上，将上面测试的对map的操作换成其它任意耗时1ms以上的操作(ex. time.Sleep(1ms))，此时mutex和channel的性能基本一致
* 在使用多核心的机器上多个goroutine写一个goroutine读的情况下，channel有一定量级的缓存的话，速度也能够和mutex基本一致

关于这个现象，我在[http://stackoverflow.com/questions/26521587/golang-how-to-share-value-message-or-mutex](http://stackoverflow.com/questions/26521587/golang-how-to-share-value-message-or-mutex)和[https://code.google.com/p/go-wiki/wiki/MutexOrChannel](https://code.google.com/p/go-wiki/wiki/MutexOrChannel)得到了解答。

golang虽然提倡用channel，但是并非“滥用”channel，真正的做法应该是在合适的场合使用合适的技术……不过看来老外也做了类似的事情\_(:з」∠)_。

比如在这次创建并发map的场合下就适合用mutex……

最后，在这次失败的实验过程中还发现了一些（没用的）东西：

* golang在大结构体的情况下，指针比复制来得快，但是在小变量（比如一个int64）的情况下，取变量地址的速度比复制来得慢。
* channel实现rwmutex类似的功能似乎还比较复杂……何况性能上似乎也不占优势所以这些事还是交给mutex好了……
* 果然channel不是万能的，只是并发编程中对锁在部分场合下的替换……