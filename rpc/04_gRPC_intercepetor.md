# 拦截器

几乎说有的 web 框架都会提供拦截器这个功能，不论是 Java 还是 python、go的 web 框架都会挺高一个拦截器，主要在于所有接口要具备的功能，比如要统计接口访问时长，验证 token，等等，我们要对所有的请求拦截一下，做一下预处理，如果在所有接口都要实现这样的功能，很明显业务接口就已经被污染了，而且不排除会有遗漏的接口。

对于 Web 框架来说一定会有一个拦截器，对于 gRPC 来说也是一样的。

在构建 gRPC 应用程序时，无论是客户端应用还是服务端应用，在远程方法执行之前或者之后，都可能需要执行一些逻辑。在 gRPC 中，可以拦截 RPC 的执行，来满足特定的需求，如日志、认证、性能度量指标等。

根据锁拦截的 RPC 调用类型，gRPC 拦截器可以分为两类。

- 一元拦截器(unary interceptor)
- 流拦截器(streaming interceptor)

在 gRPC 已经实现了拦截器的设置，我们只需要实现拦截器的逻辑。

#### 服务端拦截器

```go
package main

import (
	"context"
	"fmt"
	proto2 "maksim.website.micro/grpc_interpretor/proto"
	"net"

	"google.golang.org/grpc"
)

type Server struct {
}

func (s *Server) SayHello(ctx context.Context, request *proto2.HelloRequest) (*proto2.HelloReply, error) {
	return &proto2.HelloReply{Message: "hello " + request.Name}, nil
}
func main() {

	interceptor := func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		fmt.Println(ctx)
		fmt.Println("接收到了新的请求")
		return handler(ctx, req)
	}
	opt := grpc.UnaryInterceptor(interceptor)

	g := grpc.NewServer(opt)

	proto2.RegisterGreeterServer(g, &Server{})

	lis, err := net.Listen("tcp", "0.0.0.0:8080")
	if err != nil {
		panic("failed to listen err:" + err.Error())
	}
	err = g.Serve(lis)

	if err != nil {
		panic(err)
	}
}

```

## Client 拦截器

```golang
package main

import (
	"context"
	"fmt"
	proto2 "maksim.website.micro/grpc_interpretor/proto"
	"time"

	"google.golang.org/grpc"
)

func main() {
	interceptor := func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		start := time.Now()
		err := invoker(ctx, method, req, reply, cc, opts...)
		fmt.Printf("耗时:%s", time.Since(start))
		return err
	}
	opt := grpc.WithUnaryInterceptor(interceptor)
	conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithInsecure(), opt)
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	c := proto2.NewGreeterClient(conn)
	r, err := c.SayHello(context.Background(), &proto2.HelloRequest{Name: "Maksim"})
	if err != nil {
		panic(err)
	}
	fmt.Println(r.Message)
}

```

## 通过Metadata 和拦截器实现 gRPC 认证

为了系统的安全，当我们提供接口对外提供使用的时候，肯定要进行安全认证，比如说登录态，权限等等，这个时候我们就可以通过metadata 和 拦截器进行实现这样的功能。

以下面代码为例，我们将用户的账户密码通过 metadata 传递下去，然后使用拦截器进行验证。

```protobuf
syntax = "proto3";  //声明版本
option go_package = ".;proto";


service Greeter {
  rpc Hello(HelloRequest) returns (HelloResponse){}
}

message HelloRequest{
  string name = 1;
}

message HelloResponse {
  string name = 1;
}
```

```golang
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	proto2 "maksim.website.micro/grpc/grpc_auth/proto"
	"net"

	"google.golang.org/grpc"
)

type Server struct {
}

func (s *Server) Hello(ctx context.Context, request *proto2.HelloRequest) (*proto2.HelloResponse, error) {

	return &proto2.HelloResponse{Name: request.Name}, nil
}

func main() {
	interceptor := func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {

		md, ok := metadata.FromIncomingContext(ctx)
		fmt.Println(md)
		if !ok {
			fmt.Println("get metadata error")
			return resp, status.Error(codes.Unauthenticated, "未发现认证参数")
		}
		var username, password string
		if va1, ok := md["username"]; ok {
			username = va1[0]
		}
		if va1, ok := md["password"]; ok {
			password = va1[0]
		}
		fmt.Println(username, password)
		if username != "maksim" || password != "123456" {
			return resp, status.Error(codes.Unauthenticated, "认证未通过")
		}

		return handler(ctx, req)
	}
	opt := grpc.UnaryInterceptor(interceptor)

	g := grpc.NewServer(opt)

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



```go
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	proto2 "maksim.website.micro/grpc/grpc_auth/proto"
	"os"

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
	md := metadata.Pairs("username", "maksim", "password", "12346")
	ctx := metadata.NewOutgoingContext(context.Background(), md)
	c := proto2.NewGreeterClient(conn)
	r, err := c.Hello(ctx, &proto2.HelloRequest{Name: "Maksim"})

	if err != nil {

		st, ok := status.FromError(err)
		if !ok {
			panic(err)
		}

		fmt.Println(st.Code())
		fmt.Println(st.Message())

		os.Exit(1)
	}

	fmt.Println(r.Name)
}

```

