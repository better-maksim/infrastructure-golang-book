### 前言

gRPC默认的请求的超时时间是很长的，当你没有设置请求超时时间时，所有在运行的请求都占用大量资源且可能运行很长的时间，导致服务资源损耗过高，使得后来的请求响应过慢，甚至会引起整个进程崩溃。

为了避免这种情况，我们的服务应该设置超时时间。当客户端发起请求时候，需要传入上下文`context.Context`，用于结束`超时`或`取消`的请求。

### 客户端请求设置超时时间

0. server 端模拟超时

```go
package main

import (
	"context"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	proto2 "maksim.website.micro/grpc/grpc_guide/proto"
	"net"
	"time"

	"google.golang.org/grpc"
)

type Server struct {
}

func (s *Server) Hello(ctx context.Context, request *proto2.HelloRequest) (*proto2.HelloResponse, error) {

	time.Sleep(time.Second * 5)
	return nil, status.Error(codes.NotFound, "获取失败")
}

func main() {
	g := grpc.NewServer()
	proto2.RegisterGreeterServer(g, &Server{})
	lis, err := net.Listen("tcp", "0.0.0.0:50051")
	if err != nil {
		panic("failed to listen err:" + err.Error())
	}
	err = g.Serve(lis)

	if err != nil {
		panic(err)
	}
}

```



修改调用服务端方法

1.把超时时间设置为当前时间+3秒

```go
	ctx, _ := context.WithTimeout(context.Background(), time.Second*3)
```

2.响应错误检测中添加超时检测

```go
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"log"
	proto2 "maksim.website.micro/grpc/grpc_guide/proto"
	"time"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial("127.0.0.1:50051", grpc.WithInsecure())
	if err != nil {
		panic(err)
	}
	defer func(conn *grpc.ClientConn) {
		err := conn.Close()
		if err != nil {

		}
	}(conn)

	c := proto2.NewGreeterClient(conn)

	ctx, _ := context.WithTimeout(context.Background(), time.Second*3)
	r, err := c.Hello(ctx, &proto2.HelloRequest{Name: "Maksim"})
	if err != nil {
		st, ok := status.FromError(err)
		if ok {
			if st.Code() == codes.DeadlineExceeded {
				log.Fatalln("timeout")
			}
		}
		log.Fatalf("Call Route err: %v", err)
	}
	
	fmt.Println(r.Name)
}

```


