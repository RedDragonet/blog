---
title: Go Kit服务发现(2) Zookeeper
description:
toc: true
authors: []
tags: [go-kit]
categories: [框架]
series: []
date: 2019-06-22T20:34:53+08:00
lastmod: 2019-06-22T20:34:53+08:00
featuredVideo:
featuredImage:
draft: false
---

## 基本使用

go-kit 基于 Zookeeper 的服务发现基本使用。

<!--more-->

### 注册

```go
client, err := zk.NewClient([]string{"localhost:2181"},logger)
if err != nil{
    panic(err)
}

register := zk.NewRegistrar(client,zk.Service{
    Path: "/services/hello",
    Name: "abc",
    Data: []byte("http://127.0.0.1:8080"),
},logger)

register.Register()
```

### 发现

```go
client, err := zk.NewClient([]string{"localhost:2181"},logger)
if err != nil{
    panic(err)
}

instancer , err := zk.NewInstancer(client,"/services/hello/abc",logger)
if err != nil {
    panic(err)
}
duration := 500 * time.Millisecond
ctx := context.Background()
factory := helloFactory(ctx, "GET", "hello")
endpointer := sd.NewEndpointer(instancer, factory, logger)

endpointers,_ := endpointer.Endpoints()
```

## 底层原理

### 目录结构

```
.
├── client.go 客户端
├── client_test.go
├── doc.go
├── instancer.go 服务实例
├── instancer_test.go
├── integration_test.go
├── logwrapper.go
├── registrar.go 注册器
└── util_test.go
```

目录中主要的是这三个文件，**client.go** **instancer.go** **registrar.go**

### client.go

```go
type Client interface {
    //获取一组value通过key前缀
    GetEntries(path string) ([]string, <-chan zk.Event, error)
    //watch指定前缀的key
    CreateParentNodes(path string) error
    //注册服务
    Register(s *Service) error
    //注销服务
    Deregister(s *Service) error
    //停止zk链接
    Stop()
}


type client struct {
    *zk.Conn //组合 github.com/samuel/go-zookeeper/zk struct Conn
    clientConfig
    active bool
    quit   chan struct{}
}

func NewClient(servers []string, logger log.Logger, options ...Option) (Client, error)
func (c *client) CreateParentNodes(path string)
```

主要包含以下5个函数

- **NewClient** 创建zk客户端，client.Conn
- **GetEntries** client.Get 获取value
- **Deregister** client.Delete 删除zk中指定的key
- **CreateParentNodes** 保证key中的父节点都已经被创建。由于zk创建子节点时父节点必须都存在，如:/a/b/c,当/a/b节点不存在时，/a/b/c节点无法创建
- **Register**
  - CreateParentNodes函数创建所有父节点，已存在则跳过
  - client.CreateProtectedEphemeralSequential函数，创建一个**保护** **临时** **顺序** 节点(ProtectedEphemeralSequential)，同时将value存在此节点中。
  - 保护顺序临时节点
    - 示例:_c_**c83db041ac654566228b72cbd541bcb5**-abc0000000006，其中加粗字体为GUID，_c_为默认前缀，abc0000000006为后缀
    - **临时**节点，当zk链接会话关闭后，该节点就会被删除。
    - **顺序**节点，创建的节点的名称以GUID作为前缀。如果节点创建失败，则会发生正常的重试机制。在重试过程中，首先搜索父路径，寻找包含GUID的节点。如果找到该节点，则假定它是第一次尝试成功创建并返回给调用者的丢失节点。
    - **保护**节点,key自增后缀确保节点名称的唯一性

### registrar.go

```go
type Registrar struct {
  //zk客户端
    client  Client
    //注册的服务
    service Service
    logger  log.Logger
}

//服务的key和地址
type Service struct {
    Path string // 服务发现命名空间: /service/hello/
    Name string // 服务名称, example: abc
    Data []byte // 服务实例数据存在, 如: 10.0.2.10:80
    node string // 存储 ProtectedEphemeralSequential(保护临时顺序)节点的名称，便于Deregister函数注销服务
}

func NewRegistrar(client Client, service Service, logger log.Logger) *Registrar
func (r *Registrar) Register() 
func (r *Registrar) Deregister()
```

包含以下3个函数

- NewRegistrar 创建 Registrar
- Register 调用 **client.go**中 Register 方法
- Deregister 调用 **client.go**中 Deregister 方法

### instancer.go

```go
type Instancer struct {
  //实例缓存
    cache  *instance.Cache
    //zk客户端
    client Client
    //服务zk节点值
    path   string
    logger log.Logger
    //Instancer 主动退出 通道
    quitc  chan struct{}
}

func NewInstancer(c Client, path string, logger log.Logger) (*Instancer, error)
func (s *Instancer) loop(eventc <-chan zk.Event)
func (s *Instancer) Stop()
func (s *Instancer) Register(ch chan<- sd.Event) 
func (s *Instancer) state() sd.Event {
```

主要包含以下5个函数

- NewInstancer
  - 调用 **client.go** GetEntries函数，获取对应的一组服务地址
- loop
  - 监听服务对应的key
- Stop
  - 关闭服务监听
- Register
- Deregister