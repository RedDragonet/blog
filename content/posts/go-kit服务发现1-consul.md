---
title: Go Kit服务发现(1) Consul
description:
toc: true
authors: []
tags: [go-kit]
categories: [框架]
series: []
date: 2019-06-22T20:34:49+08:00
lastmod: 2019-06-22T20:34:49+08:00
featuredVideo:
featuredImage:
draft: false
---

## 服务发现

### 一、什么是服务？

> [OASIS](https://en.wikipedia.org/wiki/OASIS_(organization))将服务定义为“一种允许访问一个或多个功能的机制，其中使用指定的接口提供访问，并按照服务描述指定的约束和策略执行访问”。😰😰😰

<!--more-->

- 业务模块（user/mission/vip）
- 基础组件（ipdb/uuid）
- 缓存服务（redis/memcached）
- 持久化服务（Mysql/ELS/MNS）
- 网络服务(nginx/lb)
- ...

### 二、什么是服务发现？

调用方无需知道服务提供者的网络位置(ip:端口等)，只需通过服务名称(如user/item/mission)，即可调用服务

### 三、为什么需要服务发现？

> 在现代的基于云计算的微服务应用中，服务实例会被动态地分配网络地址。并且，因为自动伸缩、故障和升级，服务实例会动态地改变。故而，你的客户端代码需要用一种更加精密的服务发现机制。而不是偶尔**更新的配置文件中读取到网络地址**。

1. **场景1：** 需要新上线一个服务：
   - 提供者：配置域名、nginx、负载均衡、部署代码
   - 调用方：配置服务域名，调用具体业务
2. **场景2：** 某个热点事件的出现，导致流量爆增，需要扩容：
   - 提供者：配置nginx、负载均衡、部署代码
3. **服务发现场景1：** 需要新上线一个服务：
   - 提供者：部署代码(包含注册服务)
   - 调用方：部署代码(包含查询服务)
4. **服务发现场景2：** 某个热点事件的出现，导致流量爆增，需要扩容：
   - 提供者：部署代码（可配置自动扩容）

### 四、服务发现的流程

客户端发现提供者提供者注册中心注册中心消费者消费者注册服务健康检查查询服务提供者网络信息Ip:192.168.*.* Domain:8787访问服务

服务端发现提供者提供者注册中心注册中心负载均衡器负载均衡器消费者消费者注册服务健康检查查询服务提供者网络信息查询服务提供者网络信息转发 Ip:192.168.*.* Domain:8787Ip:192.168.*.* Domain:8787访问服务

|            | 客户端           | 服务端                     |
| :--------- | :--------------- | :------------------------- |
| 请求数     | **少一次**       | 多一次                     |
| 消费者逻辑 | 内置服务发现逻辑 | **无需客户端服务发现逻辑** |
| 业界使用   | **多一些**       | 少一些                     |

### 五、服务发现的现有解决方案

[stackshare对比页面](https://stackshare.io/stackups/consul-vs-eureka-vs-zookeeper)

|                      | ZooKeeper | Etcd      | Eureka  | Consul   | DNSSrv |
| :------------------- | :-------- | :-------- | :------ | :------- | :----- |
| 多数据中心           |           |           |         | ✅        |        |
| 自带服务发现         |           |           | ✅       | ✅        |        |
| 自带健康检查         |           |           | ✅       | ✅        |        |
| 自带WebUi            |           |           | ✅       | ✅        |        |
| 分布式Key/Value存储  | ✅         | ✅         |         | ✅        |        |
| 开源                 | ✅         | ✅         | 2.0闭源 | ✅        |        |
| 一致性               | paxos     | raft      |         | raft     |        |
| 监控                 |           | metrics   | metrics | metrics  |        |
| 使用接口(多语言能力) | 客户端    | http/grpc | http    | http/dns |        |
| CAP                  | cp        | cp        | ap      | cp       |        |
| 开发语言             | JAVA      | GO        | JAVA    | GO       |        |

## 源码原理

```
├── endpoint_cache_test.go 
├── endpoint_cache.go
├── endpointer.go
├── endpointer_test.go
├── instancer.go
├── factory.go
├── benchmark_test.go
├── registrar.go
├── doc.go
├── etcd
│   ├── client_test.go
│   ├── client.go 
│   ├── integration_test.go 
│   ├── registrar.go 
│   ├── registrar_test.go 
│   ├── example_test.go
│   ├── instancer.go  
│   ├── instancer_test.go
│   └── doc.go
├── zk
│   ├── client.go
│   ├── integration_test.go
│   ├── client_test.go
│   ├── instancer_test.go
│   ├── util_test.go
│   ├── instancer.go
│   ├── registrar.go
│   ├── logwrapper.go
│   └── doc.go
├── consul
│   ├── instancer_test.go
│   ├── instancer.go
│   ├── client_test.go
│   ├── integration_test.go
│   ├── client.go 
│   ├── registrar.go 
│   ├── registrar_test.go
│   └── doc.go
├── etcdv3
│   ├── integration_test.go
│   ├── client.go
│   ├── registrar_test.go
│   ├── registrar.go
│   ├── example_test.go
│   ├── instancer.go
│   ├── instancer_test.go
│   └── doc.go
├── lb(负载均衡)
│   ├── retry_test.go
│   ├── retry.go （多次尝试请求Endpoint）
│   ├── round_robin_test.go
│   ├── random_test.go
│   ├── round_robin.go （轮询调度Endpoint）
│   ├── random.go （随机选择Endpoint）
│   ├── balancer.go （包含一个Endpoint的接口）
│   └── doc.go
├── eureka
│   ├── util_test.go
│   ├── registrar.go
│   ├── integration_test.go
│   ├── registrar_test.go
│   ├── instancer.go
│   ├── instancer_test.go
│   └── doc.go
├── dnssrv(通过net包的dns客户端，通过SRV记录实现服务发现 [DNS SRV介绍](https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2017/03/06/dns-srv.html))
│   ├── instancer.go
│   ├── instancer_test.go
│   ├── lookup.go
│   └── doc.go
└── internal(内部通过管道实现的应用内服务发现)
    └── instance
```

# consul

## 基本使用

### 注册

```go
var client consulsd.Client
{
    consulConfig := api.DefaultConfig()
    consulConfig.Address = "localhost:8500"
    consulClient, err := api.NewClient(consulConfig)
    if err != nil {
        logger.Log("err", err)
        os.Exit(1)
    }
    client = consulsd.NewClient(consulClient)
}

check := api.AgentServiceCheck{
    HTTP:     "http://127.0.0.1:8080/health",
    Interval: "10s",
    Timeout:  "1s",
    Notes:    "基础监控检查",
}

num := rand.Intn(100) // to make service ID unique
register := consulsd.NewRegistrar(client, &api.AgentServiceRegistration{
    ID:      "hello" + strconv.Itoa(num),
    Name:    "hello",
    Tags:    []string{"hello", "hi"},
    Port:    8080,
    Address: "http://127.0.0.1",
    Check:   &check,
}, logger)

register.Register()
```

### 发现

```go
var client consulsd.Client
{
    consulConfig := api.DefaultConfig()

    consulConfig.Address = "http://localhost:8500"
    consulClient, err := api.NewClient(consulConfig)
    if err != nil {
        logger.Log("err", err)
        os.Exit(1)
    }
    client = consulsd.NewClient(consulClient)
}

tags := []string{}
passingOnly := true
duration := 500 * time.Millisecond
ctx := context.Background()
factory := helloFactory(ctx, "GET", "hello")
instancer := consulsd.NewInstancer(client, logger, "hello", tags, passingOnly)
endpointer := sd.NewEndpointer(instancer, factory, logger)
balancer := lb.NewRoundRobin(endpointer)
retry := lb.Retry(1, duration, balancer)
res, _ := retry(ctx, struct{}{})
```

## 底层原理

### 目录结构

```
.
├── client.go 
├── client_test.go
├── doc.go
├── instancer.go
├── instancer_test.go
├── integration_test.go
├── registrar.go
└── registrar_test.go
```

目录中主要的是这三个文件，**client.go** **instancer.go** **registrar.go**

### client.go

```go
// Client is a wrapper around the Consul API.
type Client interface {
    // Register a service with the local agent.
    Register(r *consul.AgentServiceRegistration) error

    // Deregister a service with the local agent.
    Deregister(r *consul.AgentServiceRegistration) error

    // Service
    Service(service, tag string, passingOnly bool, queryOpts *consul.QueryOptions) ([]*consul.ServiceEntry, *consul.QueryMeta, error)
}

type client struct {
    consul *consul.Client
}

// NewClient returns an implementation of the Client interface, wrapping a
// concrete Consul client.
func NewClient(c *consul.Client) Client {
    return &client{consul: c}
}

func (c *client) Register(r *consul.AgentServiceRegistration) error {
    return c.consul.Agent().ServiceRegister(r)
}

func (c *client) Deregister(r *consul.AgentServiceRegistration) error {
    return c.consul.Agent().ServiceDeregister(r.ID)
}

func (c *client) Service(service, tag string, passingOnly bool, queryOpts *consul.QueryOptions) ([]*consul.ServiceEntry, *consul.QueryMeta, error) {
    return c.consul.Health().Service(service, tag, passingOnly, queryOpts)
}
```

主要包含以下4个函数

- **NewClient** 创建consul客户端
- **Register** 注册服务
- **Deregister** 注销服务
- **Service** 获取服务/发现服务

### registrar.go

```go
// Registrar registers service instance liveness information to Consul.
type Registrar struct {
    client       Client
    registration *stdconsul.AgentServiceRegistration
    logger       log.Logger
}

// NewRegistrar returns a Consul Registrar acting on the provided catalog
// registration.
func NewRegistrar(client Client, r *stdconsul.AgentServiceRegistration, logger log.Logger) *Registrar {
    return &Registrar{
        client:       client,
        registration: r,
        logger:       log.With(logger, "service", r.Name, "tags", fmt.Sprint(r.Tags), "address", r.Address),
    }
}

// Register implements sd.Registrar interface.
func (p *Registrar) Register() {
    if err := p.client.Register(p.registration); err != nil {
        p.logger.Log("err", err)
    } else {
        p.logger.Log("action", "register")
    }
}

// Deregister implements sd.Registrar interface.
func (p *Registrar) Deregister() {
    if err := p.client.Deregister(p.registration); err != nil {
        p.logger.Log("err", err)
    } else {
        p.logger.Log("action", "deregister")
    }
}
```

包含以下3个函数

- NewRegistrar 创建 Registrar
- Register 调用 **client.go**中 Register 方法
- Deregister 调用 **client.go**中 Deregister 方法

### instancer.go

```go
// Instancer yields instances for a service in Consul.
type Instancer struct {
    cache       *instance.Cache
    client      Client
    logger      log.Logger
    service     string
    tags        []string
    passingOnly bool
    quitc       chan struct{}
}

// NewInstancer returns a Consul instancer that publishes instances for the
// requested service. It only returns instances for which all of the passed tags
// are present.
func NewInstancer(client Client, logger log.Logger, service string, tags []string, passingOnly bool) *Instancer 

// Stop terminates the instancer.
func (s *Instancer) Stop()

func (s *Instancer) loop(lastIndex uint64)

func (s *Instancer) getInstances(lastIndex uint64, interruptc chan struct{}) ([]string, uint64, error)

// Register implements Instancer.
func (s *Instancer) Register(ch chan<- sd.Event)

// Deregister implements Instancer.
func (s *Instancer) Deregister(ch chan<- sd.Event)

func makeInstances(entries []*consul.ServiceEntry) []string
```

主要包含以下5个函数

- NewInstancer
  - 调用 getInstances 函数，获取对应的一组服务地址，构造 Instancer 结构体
- loop
  - 循环查询调用 getInstances 函数，默认10毫秒调用一次
- Stop
  - 关闭服务监听， getInstances 函数获得一个 errStopped，停止loop
- Register
- Deregister