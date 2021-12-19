# GRPC 的四种数据流

1. 简单模式（Simple RPC）
2. 服务端数据流模式（Server-side streaming RPC）
3. 客户端数据流模式（Client-side streaming RPC）
4. 双向数据流模式（Bidirectional streaming RPC）

##### 简单模式

这种模式最为传统，即客户端发起一次请求，服务端响应一个数据，这和大家平时熟悉的RPC没有什么大的区别，所以不再详细介绍。

##### 服务端数据流模式

这种模式是客户端发起一次请求，服务端返回一段连续的数据流。典型的例子是客户端向服务端发送一个股票代码，服务端就把该股票的实时数据源源不断的返回给客户端。

##### 客户端数据流

与服务端数据流模式相反，这次是客户端源源不断的向服务端发送数据流，而在发送结束后，由服务端返回一个响应。典型的例子是物联网终端向服务器报送数据。

##### 双向数据流

顾名思义，这是客户端和服务端都可以向对方发送数据流，这个时候双方的数据可以同时互相发送，也就是可以实现实时交互。典型的例子是聊天机器人。

##### 具体实现

```protobuf
syntax = "proto3";

//声明 包名
option go_package=".;proto";

//声明grpc服务
service Greeter {
    /*
    以下 分别是 服务端 推送流， 客户端 推送流 ，双向流。
    */
    rpc GetStream (StreamReqData) returns (stream StreamResData){}
    rpc PutStream (stream StreamReqData) returns (StreamResData){}
    rpc AllStream (stream StreamReqData) returns (stream StreamResData){}
}


//stream请求结构
message StreamReqData {
    string data = 1;
}
//stream返回结构
message StreamResData {
    string data = 1;
}
```



服务端

```golang
package main

import (
	"fmt"
	"google.golang.org/grpc"
	"log"
	"net"
	"start/new_stream/proto"
	"sync"
	"time"
)

const PORT  = ":50052"

type server struct {
}

//服务端 单向流
func (s *server)GetStream(req *proto.StreamReqData, res proto.Greeter_GetStreamServer) error{
	i:= 0
	for{
		i++
		res.Send(&proto.StreamResData{Data:fmt.Sprintf("%v",time.Now().Unix())})
		time.Sleep(1*time.Second)
		if i >10 {
			break
		}
	}
	return nil
}

//客户端 单向流
func (s *server) PutStream(cliStr proto.Greeter_PutStreamServer) error {

	for {
		if tem, err := cliStr.Recv(); err == nil {
			log.Println(tem)
		} else {
			log.Println("break, err :", err)
			break
		}
	}

	return nil
}

//客户端服务端 双向流
func(s *server) AllStream(allStr proto.Greeter_AllStreamServer) error {

	wg := sync.WaitGroup{}
	wg.Add(2)
	go func() {
		for {
			data, _ := allStr.Recv()
			log.Println(data)
		}
		wg.Done()
	}()

	go func() {
		for {
			allStr.Send(&proto.StreamResData{Data:"ssss"})
			time.Sleep(time.Second)
		}
		wg.Done()
	}()

	wg.Wait()
	return nil
}

func main(){
	//监听端口
	lis,err := net.Listen("tcp",PORT)
	if err != nil{
		panic(err)
		return
	}
	//创建一个grpc 服务器
	s := grpc.NewServer()
	//注册事件
	proto.RegisterGreeterServer(s,&server{})
	//处理链接
	err = s.Serve(lis)
	if err != nil {
		panic(err)
	}
}
```

客户端

```golang
package main

import (
	"google.golang.org/grpc"

	"context"
	_ "google.golang.org/grpc/balancer/grpclb"
	"log"
	"start/new_stream/proto"
	"time"
)

const (
	ADDRESS = "localhost:50052"
)


func main(){
	//通过grpc 库 建立一个连接
	conn ,err := grpc.Dial(ADDRESS,grpc.WithInsecure())
	if err != nil{
		return
	}
	defer conn.Close()
	//通过刚刚的连接 生成一个client对象。
	c := proto.NewGreeterClient(conn)
	//调用服务端推送流
	reqstreamData := &proto.StreamReqData{Data:"aaa"}
	res,_ := c.GetStream(context.Background(),reqstreamData)
	for {
		aa,err := res.Recv()
		if err != nil {
			log.Println(err)
			break
		}
		log.Println(aa)
	}
	//客户端 推送 流
	putRes, _ := c.PutStream(context.Background())
	i := 1
	for {
		i++
		putRes.Send(&proto.StreamReqData{Data:"ss"})
		time.Sleep(time.Second)
		if i > 10 {
			break
		}
	}
	//服务端 客户端 双向流
	allStr,_ := c.AllStream(context.Background())
	go func() {
		for {
			data,_ := allStr.Recv()
			log.Println(data)
		}
	}()

	go func() {
		for {
			allStr.Send(&proto.StreamReqData{Data:"ssss"})
			time.Sleep(time.Second)
		}
	}()

	select {
	}

}
```



代码地址：





