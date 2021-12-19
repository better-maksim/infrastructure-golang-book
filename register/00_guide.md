## Naocs

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您实现动态服务发现、服务配置管理、服务及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构(例如微服务范式、云原生范式)的服务基础设施。

#### Nacos 的安装

推荐使用 Docker 一键部署，这样省去了很多麻烦的事情。

```shell
git clone https://github.com/nacos-group/nacos-docker.git

cd nacos-docker

docker-compose -f example/standalone-derby.yaml up
```

安装指定版本

修改 build 文件夹中的 Dockerfile 即可

## Nacos 配置中心

```go
package main

import (
	"fmt"
	"github.com/nacos-group/nacos-sdk-go/clients"
	"github.com/nacos-group/nacos-sdk-go/common/constant"
	"github.com/nacos-group/nacos-sdk-go/vo"
	"time"
)

func main() {
	sc := []constant.ServerConfig{
		{
			IpAddr: "127.0.0.1",
			Port:   8848,
		},
	}

	cc := constant.ClientConfig{
		TimeoutMs:           5000,
		NamespaceId:         "0dbffc20-5d6f-4b1f-9c76-c2b6d71c5e46",
		NotLoadCacheAtStart: true,
		LogDir:              "/tmp/nacos/log",
		CacheDir:            "/tmp/nacos/cache",
		RotateTime:          "1h",
		MaxAge:              3,
		LogLevel:            "debug",
	}

	configClient, err := clients.CreateConfigClient(map[string]interface{}{
		"serverConfigs": sc,
		"clientConfig":  cc,
	})
	if err != nil {
		panic(err)
	}

	content, err := configClient.GetConfig(vo.ConfigParam{DataId: "nacos-test-guozhu", Group: "dev"})

	if err != nil {
		panic(err)
	}

	fmt.Println(content)

	err = configClient.ListenConfig(vo.ConfigParam{
		DataId: "guozhu-test",
		Group:  "dev",
		OnChange: func(namespace, group, dataId, data string) {
			fmt.Println("group:" + group + ", dataId:" + dataId + ", data:" + data)
		},
	})
	if err != nil {
		panic(err)
	}

	time.Sleep(3000 * time.Second)
}
```

## Nacos 注册中心

### 什么是注册中心

**服务注册中心:**用来实现微服务实例的自动注册与发现，是分布式系统中的核心基础服务 

![image-20211214134704050](/Users/guozhu/Documents/micro-service/_media/image-20211214134704050.png)



![image-20211214134714313](/Users/guozhu/Documents/micro-service/_media/image-20211214134714313.png)



![image-20211214134733784](/Users/guozhu/Documents/micro-service/_media/image-20211214134733784.png)

![image-20211214134752569](/Users/guozhu/Documents/micro-service/_media/image-20211214134752569.png)

![image-20211214134806704](/Users/guozhu/Documents/micro-service/_media/image-20211214134806704.png)

![image-20211214134821472](/Users/guozhu/Documents/micro-service/_media/image-20211214134821472.png)

### go 向 Nacos 注册

- 注册

- 注销

- 健康检查

  

