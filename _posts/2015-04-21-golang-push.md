---
layout: post
title: golang做push系统遇到的问题的整理(一)
categories: [golang]
tags: [golang,记录]
---
啊哈哈哈，写着玩……_(:з」∠)_

golang版本1.3,最新的是1.4的。吧。大概。操作系统10.9.4的osx。

只是做一个测试，server端和client端代码都很简单。

服务端代码：
```
package server

import (
	"net"
	"fmt"
	"sync"
)

type Server struct{
	ip       string
	port     string
	listener net.Listener
	clients  []net.Conn
	running  bool

	counter int64
	lock    sync.Mutex
}

func NewServer(ip string, port string) Server {
	s := Server{}
	s.ip = ip
	s.port = port
	s.clients = make([]net.Conn, 100000, 100000)
	s.counter = 0
	fmt.Printf("set a lock")
	return s
}

func (s *Server) count() {
	fmt.Printf("now has clients %d \n", s.counter)
}

func (s *Server) Start() {
	listener, err := net.Listen("tcp", s.ip+":"+s.port)
	if err != nil {
		fmt.Printf("error: %v", err)
	}
	s.listener = listener
	go s.accept()
}

func (s *Server) addCount() {
	s.lock.Lock()
	defer s.lock.Unlock()
	s.counter ++
}

func (s *Server) accept() {
	for {
		client, err := s.listener.Accept()
		
		if err != nil {
			fmt.Printf("error: %v", err)
		}

		s.clients[s.counter] = client
		s.addCount()
		if s.counter%1000 == 0 {
			s.count()
		}

		a := "hello world"
		client.Write([]byte(a))
	}
}
```

至于执行嘛就是

```
s := server.NewServer(ip, strconv.Itoa(port))
s.Start()
```

客户端代码：

```
package main

import (
	"fmt"
	"net"
	tomb "gopkg.in/tomb.v1"
	"flag"
	"strconv"
)

var (
	ip string
	port int
)

//only for test
func main() {
	flag.StringVar(&ip, "ip", "127.0.0.1", "server listene ip")
	flag.IntVar(&port, "port", 9987, "server listen port")
	flag.Parse()
	loop := 15000
	errLoop := 0
	for i := 0; i < loop; i ++ {
		conn , err := net.Dial("tcp", ip+":"+strconv.Itoa(port))
		if err != nil {
			if errLoop%1000 == 0 {
				fmt.Printf("error: %v \n", err)
			}
			errLoop++
		}
		if i%1000 == 0 {
			fmt.Println(i)
			go func() {
				bytes := make([]byte, 100, 100)
				conn.Read(bytes)
				fmt.Println(string(bytes))
			}()
		}
	}

	tomb1 := tomb.Tomb{}
	tomb1.Wait()
}
```

没做任何容错，能跑就行。恩。

在迈向1万个链接的路上很快就遇到了第一个问题，提示“too many open files”。神机妙算早已知晓，但是改起来却很麻烦……

本来以为只要单纯的```sudo ulimit -n [num]```就可以搞定问题，但是只要我想设置超出10240的数字，就会报错```-bash: ulimit: open files: cannot modify limit: Operation not permitted```，很残念的是sudo对此无效，它只会临时创建一个root环境，执行完没有任何效果。

还好谷歌到解决方案：在/etc/launchd.conf文件中新增一行```limit maxfiles 1000000 1000000```。之前intellij安装0.95的golang插件时也需要往这里面写配置```setenv GOPATH $(go env GOPATH)
  launchctl setenv GOROOT $(go env GOROOT)```

于是正常创建1w个链接。继续准备创建到4w个链接。这次居然又出问题了。大概创建到1w6的时候客户端提示错误"Can't assign requested address"。

在[这里](http://superuser.com/questions/145989/does-mac-os-x-throttle-the-rate-of-socket-creation)找到了问题原因和解决方案。

按照文件的提示设置了参数```sudo sysctl -w net.inet.ip.portrange.first=39192```，这下可以多链接1w个客户端链接了。看了看系统占用，内存和cpu都是毫无压力，很好。

其实就是客户端链接服务端的时候会占用一个端口号。本来我一直以为服务端accept到一个请求后创建到客户端的链接也会产生一个端口号，实际上是只有客户端会占用一个端口号，客户端用这个端口号继续和服务端的listen端口做通讯。

本来按照socket的[server_ip:server_port <---> client_ip:client_port]的四元组理论来说，如果我这个时候还想再用本机生成客户端链接的话，只要再启动一个server端，更换下它的监听port号就可以了，但很残念我发现这么做没能提升客户端连接数。

我猜想是不是需要加上socket的so_reuseaddr参数。SO_REUSEADDR可以用在以下四种情况下。
(摘自《Unix网络编程》卷一，即UNPv1)

1.当有一个有相同本地地址和端口的socket1处于TIME_WAIT状态时，而你启
动的程序的socket2要占用该地址和端口，你的程序就要用到该选项。
2.SO_REUSEADDR允许同一port上启动同一服务器的多个实例(多个进程)。但
每个实例绑定的IP地址是不能相同的。在有多块网卡或用IP Alias技术的机器可
以测试这种情况。
3.SO_REUSEADDR允许单个进程绑定相同的端口到多个socket上，但每个soc
ket绑定的ip地址不同。这和2很相似，区别请看UNPv1。
4.SO_REUSEADDR允许完全相同的地址和端口的重复绑定。但这只用于UDP的
多播，不用于TCP。

然后我发现我没找到golang哪里设置这么些参数……

后来在[googlegroup](https://groups.google.com/forum/#!topic/golang-nuts/0ExU8aBcNVI)(自带楼梯)上发现解释说，似乎现在只能用golang的syscal包的函数实现。

不过管他的，我还有伟大docker可用，我决定过段时间拿docker开几个虚拟机做压测，就可以避免单机的端口数不够用的问题了……

最后的最后记录一个发现，我最后做得设置，大概本机测试时可以创建将近4w个链接。我如果开一个server端连上2w个客户端链接，此时再开一个server端监听另一个端口，让客户端连过去，这个时候macos的内核进程(kernel_task)cpu占用率位100%……