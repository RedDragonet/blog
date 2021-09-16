---
title: Go Kit服务发现(4) Eureka
description:
toc: true
authors: []
tags: [go-kit]
categories: [框架]
series: []
date: 2019-06-22T20:35:01+08:00
lastmod: 2019-06-22T20:35:01+08:00
featuredVideo:
featuredImage:
draft: false
---

## 基本使用

go-kit 基于 Eureka 的服务发现基本使用。

<!--more-->

### 注册

```go
logger := log.NewLogfmtLogger(os.Stdout)
var fargoConfig fargo.Config
fargoConfig.Eureka.ServiceUrls = []string{"http://localhost:8761/eureka"}
// 订阅服务器应轮询更新的频率。
fargoConfig.Eureka.PollIntervalSeconds = 1

instance := &fargo.Instance{
    InstanceId : "实例ID",
    //HostName:         "127.0.0.1",
    Port:   8080,
    App:    "hello",
    IPAddr: "http://127.0.0.1",
    //HealthCheckUrl: "http://localhost:8080/hello",
    //StatusPageUrl:  "http://localhost:8080/hello",
    //HomePageUrl:    "http://localhost:8080/hello",
    Status:         fargo.UP,
    DataCenterInfo: fargo.DataCenterInfo{Name: fargo.MyOwn},
    LeaseInfo:      fargo.LeaseInfo{RenewalIntervalInSecs: 1},
}
fargoConnection := fargo.NewConnFromConfig(fargoConfig)
register := eureka.NewRegistrar(&fargoConnection, instance, logger)

register.Register()
defer register.Deregister()
```

### 发现

```go
logger := log.NewLogfmtLogger(os.Stdout)
var fargoConfig fargo.Config
fargoConfig.Eureka.ServiceUrls = []string{"http://localhost:8761/eureka"}
fargoConfig.Eureka.PollIntervalSeconds = 1

fargoConnection := fargo.NewConnFromConfig(fargoConfig)
instancer := eureka.NewInstancer(&fargoConnection,"hello",logger)
```

## 底层原理

### 目录结构

```
.
├── doc.go
├── instancer.go 服务实例 
├── instancer_test.go
├── integration_test.go
├── registrar.go 注册器
├── registrar_test.go
└── util_test.go
```

目录中主要的是这三个文件，**instancer.go** **registrar.go**
与sd目录下的其他注册发现组件不同，[作者删除了client.go文件](https://github.com/go-kit/kit/commit/443f6ead51a91ddf48136543c1878aedb3efb2fd#diff-1bc8b1643e279c1d7764b3ac84134584)，改为使用[fargo.EurekaConnection](https://github.com/hudl/fargo/blob/master/struct.go#L18)

### registrar.go

```go
// Registrar maintains service instance liveness information in Eureka.
type Registrar struct {
    conn     fargoConnection
    instance *fargo.Instance
    logger   log.Logger
    quitc    chan chan struct{}
    sync.Mutex
}

func NewRegistrar(conn fargoConnection, instance *fargo.Instance, logger log.Logger) *Registrar
func (r *Registrar) Register()
func (r *Registrar) Deregister()
func (r *Registrar) loop()
func (r *Registrar) heartbeat() error 
```

包含以下3个函数

- **NewRegistrar** 创建 Registrar
- **Register** 通过 fargoConnection 注册服务
- **Deregister** 通过 fargoConnection 注销服务
- **loop**
  - `fargoConfig.Eureka.PollIntervalSeconds` 设置的定时器，每隔设定的秒数(默认30秒)，调用**heartbeat**函数
  - 监听`Registrar.quitc`,退出loop
- **heartbeat**
  - 调用`fargo.EurekaConnection.HeartBeatInstance`，向Eureka中注册的服务实例发送心跳请求，具体的发送方式为向指定的节点发送Http PUT请求，如`PUT http://Eureka Server(s):8761/eureka/apps/APP名称/实例ID`，请求返回HTTP状态码为200即为成功

### instancer.go

```go
type Instancer struct {
    cache  *instance.Cache
    conn   fargoConnection
    app    string
    logger log.Logger
    quitc  chan chan struct{}
}
```