# GRPC 健康检查



对于 gRPC 来说，其对外暴露的是一个 proto 的协议，所以说对于 grpc 来说需要一个更加好的规范，这个规范是由 gRPC 制定的，在最开始的时候，gRPC 就预留了接口，任何一个第三方的注册中心想要进行健康检查，就可以在内部将 gRPC 事先定义好的 proto 文件纳入进去，然后通过 proto 文件发起请求。



规范：[grpc/health-checking.md at master · grpc/grpc · GitHub](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)



这个协议里面明确的说明，如果第三方想要做健康检查就发起这个请求就可以了，这个请求里面我们可以看一下：



在这个 proto 里面，需要实现 headlth 和 Watch。

在 gRPC 里面已经帮我们实现了这两个方法。



