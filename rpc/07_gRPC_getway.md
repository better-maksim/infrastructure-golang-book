# 前言

我们通常把 `RPC` 用作内部通信，而使用 `Restful Api` 进行外部通信。为了避免写两套应用，我们使用[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)把`gRPC`转成`HTTP`。服务接收到 `HTTP` 请求后，`grpc-gateway` 把它转成 `gRPC` 进行处理，然后以 `JSON` 形式返回数据。本篇代码以上篇为基础，最终转成的`Restful Api`支持`bearer token`验证、数据验证，并添加`swagger`文档。

[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)是protoc的一个插件。它读取gRPC服务定义，并生成一个反向代理服务器，将RESTful JSON API转换为gRPC。此服务器是根据gRPC定义中的自定义选项生成的。

官网：[gRPC-Gateway | gRPC-Gateway Documentation Website (grpc-ecosystem.github.io)](https://grpc-ecosystem.github.io/grpc-gateway/)

## 环境准备

| 工具                    | 介绍                                             |
| ----------------------- | ------------------------------------------------ |
| protobuf                | protocol buffer 编译所需的命令行                 |
| protoc-gen-go           | 从 proto 文件，生成 .go 文件                     |
| protoc-gen-go-grpc      | 从 proto 文件，生成 GRPC 相关的 .go 文件         |
| protoc-gen-grpc-gateway | 从 proto 文件，生成 grpc-gateway 相关的 .go 文件 |
| protoc-gen-openapiv2    | 从 proto 文件，生成 swagger 界面所需的参数文件   |

```shell
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway  # gateway 插件安装
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger  #自动生成 api 文档
```

Proto-path 环境

hub.fastgit.org

```
git clone https://github.com/googleapis/googleapis
git clone https://github.com/grpc-ecosystem/grpc-gateway.git
git clone https://github.com/envoyproxy/protoc-gen-validate.git
```

将已经下载好的仓库移动到 /usr/lcoal/include

5、编写protot文件并编译
5.1 生成swagger API 文档，需要用到google/api中http.proto、annotations.proto这两个文件，先拷贝到项目目录下

进入项目的根目录

```
$ cd /path/to/project
$ mkdir -p ./proto/google/api
$ cp -r $GOPATH/pkg/mod/github.com/grpc-ecosystem/grpc-gateway@v1.14.4/third_party/googleapis/google/api/* ./proto/google/api
```



5.2 编写protot文件

```protobuf
syntax = "proto3";

option go_package = ".;proto";

import "google/api/annotations.proto";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      get: "/hello_world"
    };
  };
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}


```



5. 使用protoc命令编译生成go源码及swagger.json

```shell
protoc helloworld.proto --proto_path=. --proto_path=/usr/local/include/googleapis --proto_path=/usr/local/include/grpc-gateway --grpc-gateway_out . --swagger_out=logtostderr=true:. --go_out=plugins=grpc:.
```

![image-20211212211031031](/Users/guozhu/Library/Application Support/typora-user-images/image-20211212211031031.png)

6、使用docker启动swagger ui服务，查看生成的API文档

```shell
docker run --rm -d -p 80:8080 -e SWAGGER_JSON=/foo/hello.swagger.json -v /path/to/project/proto:/foo swaggerapi/swagger-ui
```

docker run --rm -d -p 80:8080 -e SWAGGER_JSON=/foo/hello.swagger.json /Users/guozhu/Workspace/rrpc/grpc/grpc_gateway/proto:/foo swaggerapi/swagger-ui

访问swagger ui服务所在的主机的80端口，即可查看API文档





7、编写server.go和client.go调用RPC服务
7.1 server.go

```go
package main

import (
	"context"
	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
	"log"
	proto2 "maksim.website.micro/grpc/gateway/proto"
	"net"
	"net/http"
)


type Server struct {
}

func (s *Server) SayHello(ctx context.Context, request *proto2.HelloRequest) (*proto2.HelloReply, error) {
	return &proto2.HelloReply{Message: "hello " + request.Name}, nil
}

func main() {
	go startGRPCGateWay()
	lis, err := net.Listen("tcp", ":8081")
	if err != nil {
		log.Fatalf("faliled to listen: %v", err)
	}
	g := grpc.NewServer()
	proto2.RegisterGreeterServer(g, &Server {})
	err = g.Serve(lis)

	if err !=nil {
		panic(err)
	}

}

func startGRPCGateWay() {
	c := context.Background()
	c, cancel := context.WithCancel(c)
	mux := runtime.NewServeMux()
	defer cancel()
	err := proto2.RegisterGreeterHandlerFromEndpoint(c,
		mux,
		":8081",
		[]grpc.DialOption{grpc.WithInsecure()},
	)
	if err != nil {
		log.Fatalf("cannot start grpc gateway: %v", err)
	}
	err = http.ListenAndServe(":8080", mux)
	if err != nil {
		log.Fatalf("cannot start grpc gateway: %v", err)
	}

}

```


7.2 client.go

```golang
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
	proto2 "maksim.website.micro/grpc/grpc_gateway/proto"
)

func main() {
	conn, err := grpc.Dial("127.0.0.1:8081", grpc.WithInsecure())
	if err != nil {
		panic(err)
	}
	defer func(conn *grpc.ClientConn) {
		err := conn.Close()
		if err != nil {
			
		}
	}(conn)

	c := proto2.NewGreeterClient(conn)
	r, err := c.SayHello(context.Background(), &proto2.HelloRequest{Name: "Maksim"})
	if err != nil {
		panic(err)
	}
	fmt.Println(r.Message)

}
```

7.3 测试效果



grpc client 连接

![image-20211212205959672](/Users/guozhu/Library/Application Support/typora-user-images/image-20211212205959672.png)

curl 请求

```shell
 curl --location --request GET 'localhost:8080/hello_world?name=curl'
```

![image-20211212210058160](/Users/guozhu/Library/Application Support/typora-user-images/image-20211212210058160.png)



# 验证器

proto 虽然写起来比较麻烦，但是 proto 就相当于是一个文档，我们只需要看 proto 就可以知道我们所提供的接口的具体信息（接口名，参数类型等等），虽然有了这样的效果，但是我们在进行接口开发的时候，对于我们来说用户发起请求之前还有一个表单验证。

对于 proto 可以做到一个类型的限制，但是配置并不详细，所以我们需要一个表单验证的功能，来判断请求是是否符合我们的配置。

我们可以使用 [GitHub - envoyproxy/protoc-gen-validate: protoc plugin to generate polyglot message validators (fastgit.org)](https://hub.fastgit.org/envoyproxy/protoc-gen-validate) 来实现验证器的功能。

下面我们来对原有 proto 进行更改

```golang
syntax = "proto3";
option go_package = ".;proto";

import "google/api/annotations.proto";
import "validate/validate.proto";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      get: "/hello_world"
    };
  };
}

message HelloRequest {
  string name = 1 [(validate.rules).string.email = true];;
}

message HelloReply {
  string message = 1;
}

```

重新生成

```shell
protoc helloworld.proto  \
--proto_path=. --proto_path=/usr/local/include/protoc-gen-validate  \
--proto_path=/usr/local/include/googleapis \
--proto_path=/usr/local/include/grpc-gateway \
--grpc-gateway_out . \
--swagger_out=logtostderr=true:.\
--go_out=plugins=grpc:. \
--validate_out="lang=go:."
```

