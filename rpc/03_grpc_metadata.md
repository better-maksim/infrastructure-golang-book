
# metadata

## metadata 简介

gRPC 让我们可以像调本地调用一样实现远程调用，对于每一次的 RPC 调用中，都可能会有一些有用的数据，而这些数据就可以通过 metadata 来传递。metadata 是以 key-value 的形式存储数据的，其中 key 是 string 类型，而 value 是[]string，即一个字符串数组类型。

**metadata 使得 client 和 server 能够为对方提供关于本次调用的一些信息**，就像是一次 http 请求的 header ，http 中 header 的生命周期是一次 http 请求，那么 metadata 的生命周期就是一次 RPC 调用。

metadata 的源代码地址：[grpc-go/metadata at master · grpc/grpc-go · GitHub](https://github.com/grpc/grpc-go/tree/master/metadata)

## 在 Go 中使用 metadata

md 的类型实际上是 map，key 是 string，value 是 string 类型的 slice。

```go
// MD is a mapping from metadata keys to values. Users should use the following
// two convenience functions New and Pairs to generate MD.
type MD map[string][]string
```

创建的时候可以像创建普通 map 类型一样使用 new 关键字

```golang
// 第一种方式
md := metadata.New(map[string]string{"key1": "val1", "key2": "val2"})
// 第二种方式
md := metadata.Pairs( "key1","val1",  "key2","value2")
```

需要注意的是第二种方式，key 不区分大小写，会被统一转换为小写，下面是 `Pairs`的源代码。

```golang
func Pairs(kv ...string) MD {
	if len(kv) % 2 == 1 {
		panic(fmt.Sprintf("metadata: Pairs got the odd number of input pairs for metadata: %d", len(kv)))
	}
	md := MD{}
	for i := 0; i < len(kv); i += 2 {
		key := strings.ToLower(kv[i])
		md[key] = append(md[key], kv[i+1])
	}
	return md
}
```

准备 proto

```protobuf
syntax = "proto3";

package metadata_pb;

service Greeter {
  rpc Hello(HelloRequest) returns (HelloResponse){}
}

message HelloRequest{
  string name = 1;
}

message HelloResponse {
  string greeting = 1;
}
```

发送 metadata

```go
func main() {
	conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithInsecure())
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	md := metadata.Pairs("key1","val1","key2","value2")

	//新建一个有 metadata 的 context
	ctx := metadata.NewOutgoingContext(context.Background(), md)

	c := metadata_pb.NewGreeterClient(conn)
	r, err := c.Hello(ctx, &metadata_pb.HelloRequest{Name: "Maksim"})
	if err != nil {
		panic(err)
	}
	fmt.Println(r)
}
```

服务端接收 metadata

```GO

type GreeterService struct {
  
}

func (s *GreeterService) Hello(ctx context.Context, in *metadata_pb.HelloRequest) (*metadata_pb.HelloResponse, error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		log.Fatalf("获取 metadata 失败")
	}
	fmt.Println(md)
	return &metadata_pb.HelloResponse{Greeting: "Hello " + in.Name}, nil
}



func main() {

	lis, err := net.Listen("tcp", fmt.Sprintf("%s:%d", "127.0.0.1", 8080))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer() //起一个服务

	metadata_pb.RegisterGreeterServer(s, &GreeterService{})

	// 注册反射服务 这个服务是CLI使用的 跟服务本身没有关系
	reflection.Register(s)

	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}

}
```

执行代码后可以看到输出

![image-20211208200250541](/Users/guozhu/Library/Application Support/typora-user-images/image-20211208200250541.png)

