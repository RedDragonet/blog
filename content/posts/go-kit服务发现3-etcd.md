---
title: Go Kit服务发现(3) Etcd
description:
toc: true
authors: []
tags: [go-kit]
categories: [框架]
series: []
date: 2019-06-22T20:34:57+08:00
lastmod: 2019-06-22T20:34:57+08:00
featuredVideo:
featuredImage:
draft: false
---

## 基本使用

go-kit 基于 Etcd 的服务发现基本使用。

<!--more-->

### 注册

```go
//Etcd客户端
client, _ := etcdv3.NewClient(context.Background(),[]string{"http://localhost:2379"},etcdv3.ClientOptions{})

//注册器
register := etcdv3.NewRegistrar(client,etcdv3.Service{
    Key: "/services/hello/",
    Value: "http://127.0.0.1:8080",
},logger)
register.Register()
defer register.Deregister()
```

### 发现

```go
//Etcd客户端
client,_ := etcdv3.NewClient(context.Background(),[]string{"http://localhost:2379"},etcdv3.ClientOptions{})

//服务实例
instancer , _ := etcdv3.NewInstancer(client,"/services/hello/",logger)
```

## 底层原理

### 目录结构

```
.
├── client.go 客户端
├── doc.go
├── example_test.go
├── instancer.go 服务实例
├── instancer_test.go
├── integration_test.go
├── registrar.go 注册器
└── registrar_test.go
```

目录中主要的是这三个文件，**client.go** **instancer.go** **registrar.go**

### client.go

```go
type Client interface {
    //获取一组value通过key前缀
    GetEntries(prefix string) ([]string, error)
    //watch指定前缀的key
    WatchPrefix(prefix string, ch chan struct{})
    //注册服务
    Register(s Service) error
    //注销服务
    Deregister(s Service) error
    //etcd 
    LeaseID() int64
}

type client struct {
    //etcd客户端使用v3版本api
    cli *clientv3.Client
    ctx context.Context
    //etcd key/value 操作实例
    kv clientv3.KV
    // etcd watcher 操作实例
    watcher clientv3.Watcher
    // watcher context
    wctx context.Context
    // watcher cancel func
    wcf context.CancelFunc
    // leaseID will be 0 (clientv3.NoLease) if a lease was not created
    leaseID clientv3.LeaseID

    //etcdKeepAlive实现心跳检测
    hbch <-chan *clientv3.LeaseKeepAliveResponse
    // etcd Lease 操作实例
    leaser clientv3.Lease
}
func NewClient(ctx context.Context, machines []string, options ClientOptions) (Client, error)
```

主要包含以下6个函数

- **NewClient** 创建etcd客户端，赋值给 client.cli
- **GetEntries** 通过 client.kv 获取value
- **WatchPrefix** 通过 client.watcher 监听key
- **Deregister** 通过 client.cli 服务绑定的key
- **LeaseID** return client.leaseID
- Register
  - 初始化 client.leaser
  - 初始化 client.watcher
  - 初始化 client.kv
  - 通过 client.kv 操作写入etcd，服务注册的key和value
  - 创建 client.leaseID，默认心跳3秒，lease TTL9秒
  - client.leaser调用KeepAlive

### registrar.go

```go
type Registrar struct {
    //etcd客户端
    client  Client
    //注册的服务
    service Service
    logger  log.Logger

    //服务Deregister并发锁
    quitmtx sync.Mutex
    //服务退出通道
    quit    chan struct{}
}

//服务的key和地址
type Service struct {
    Key   string // unique key, e.g. "/service/foobar/1.2.3.4:8080"
    Value string // returned to subscribers, e.g. "http://1.2.3.4:8080"
    TTL   *TTLOption
}

//服务心跳检测
type TTLOption struct {
    heartbeat time.Duration // e.g. time.Second * 3
    ttl       time.Duration // e.g. time.Second * 10
}

func NewTTLOption(heartbeat, ttl time.Duration) *TTLOption 
func NewRegistrar(client Client, service Service, logger log.Logger) *Registrar
func (r *Registrar) Register()
func (r *Registrar) Deregister()
```

主要包含以下4个函数

- NewTTLOption 心跳检测参数
- NewRegistrar 创建 Registrar
- Register 调用 **client.go**中 Register 方法
- Deregister 调用 **client.go**中 Deregister 方法

### instancer.go

```go
type Instancer struct {
  //实例缓存
    cache  *instance.Cache
    //etcd客户端
    client Client
    //实例前缀
    prefix string
    logger log.Logger
    //Instancer 主动退出 通道
    quitc  chan struct{}
}

func NewInstancer(c Client, prefix string, logger log.Logger) (*Instancer, error) 
func (s *Instancer) loop() 
func (s *Instancer) Stop()
func (s *Instancer) Register(ch chan<- sd.Event) 
func (s *Instancer) Deregister(ch chan<- sd.Event)
```

主要包含以下5个函数

- NewInstancer
  - 调用 **client.go** GetEntries函数，获取对应的一组服务地址
  - 查询到的服务地址写入缓存 Instancer.cache
- loop
  - 监听服务对应的key
- Stop
  - 关闭服务监听
- Register
- Deregister