# 配置中心在分布式架构中的作用

可以在不重新编译代码的情况下，改变程序运行逻辑、调整边界值、被调用模块路由信息等等，用来方便维护，提高工作效率一种手段

## 常见的形式

- xml
- yaml
- conf
- properties

yml 例子：

```yml
server: 
	udp:
		port:58885 
		recivebuf:16777216 
		sendbuf:16777216
	isControl:true
```

properties：

```properties
server.udp.port=58885 
server.udp.recivebuf=16777216 
server.udp.sendbuf=16777216 
server.isControl=true
```

## 本地配置的缺点

- 重启生效
- 不易维护
- 生效慢

![image-20211213164357501](/Users/guozhu/Library/Application Support/typora-user-images/image-20211213164357501.png)

## 功能预期

- 集中管理
- 批量操作
- 热更新

## 什么是配置中心

**配置中心：**分布式系统中集中化管理线上应用程序配置项的管理中心，引入配置中心后带来的收益：

- 效率提升
- 维护成本降低
- 安全性提升



