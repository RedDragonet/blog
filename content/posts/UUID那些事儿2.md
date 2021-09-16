---
title: UUID那些事儿（中）
description:
toc: true
authors: []
tags: [uuid]
categories: [杂谈]
series: []
date: 2017-12-10T20:10:04+08:00
lastmod: 2017-12-10T20:10:04+08:00
featuredVideo:
featuredImage:
draft: false
---

不遵守UUID协议规范的唯一识别码生成

<!--more-->

# twitter/snowflake

【此节转载】

## Snowflake算法核心

把时间戳，工作机器id，序列号组合在一起。
![snowflake-64bit](http://static.tongzhao.red/img/20180123204222.jpg)
除了最高位bit标记为不可用以外，其余三组bit占位均可浮动，看具体的业务需求而定。默认情况下41bit的时间戳可以支持该算法使用到2082年，10bit的工作机器id可以支持1023台机器，序列号支持1毫秒产生4095个自增序列id。下文会具体分析。

## Snowflake – 时间戳

这里时间戳的细度是毫秒级，具体代码如下，建议使用64位linux系统机器，因为有vdso，gettimeofday()在用户态就可以完成操作，减少了进入内核态的损耗。

```c
uint64_t generateStamp()
{
    timeval tv;
    gettimeofday(&tv, 0);
    return (uint64_t)tv.tv_sec * 1000 + (uint64_t)tv.tv_usec / 1000;
}
```

默认情况下有41个bit可以供使用，那么一共有T（1llu << 41）毫秒供你使用分配，年份 = T / (3600 * 24 * 365 * 1000) = 69.7年。如果你只给时间戳分配39个bit使用，那么根据同样的算法最后年份 = 17.4年。

## Snowflake – 工作机器id

严格意义上来说这个bit段的使用可以是进程级，机器级的话你可以使用MAC地址来唯一标示工作机器，工作进程级可以使用IP+Path来区分工作进程。如果工作机器比较少，可以使用配置文件来设置这个id是一个不错的选择，如果机器过多配置文件的维护是一个灾难性的事情。

这里的解决方案是需要一个工作id分配的进程，可以使用自己编写一个简单进程来记录分配id，或者利用Mysql auto_increment机制也可以达到效果。
![snowflake-工作id](http://static.tongzhao.red/img/20180123204222.jpg)

工作进程与工作id分配器只是在工作进程启动的时候交互一次，然后工作进程可以自行将分配的id数据落文件，下一次启动直接读取文件里的id使用。

PS：这个工作机器id的bit段也可以进一步拆分，比如用前5个bit标记进程id，后5个bit标记线程id之类

## Snowflake – 序列号

序列号就是一系列的自增id（多线程建议使用atomic），为了处理在同一毫秒内需要给多条消息分配id，若同一毫秒把序列号用完了，则“等待至下一毫秒”。

```c
uint64_t waitNextMs(uint64_t lastStamp)
{
    uint64_t cur = 0;
    do {
        cur = generateStamp();
    } while (cur <= lastStamp);
    return cur;
}
```

总体来说，是一个很高效很方便的GUID产生算法，一个int64_t字段就可以胜任，不像现在主流128bit的GUID算法，即使无法保证严格的id序列性，但是对于特定的业务，比如用做游戏服务器端的GUID产生会很方便。另外，在多线程的环境下，序列号使用atomic可以在代码实现上有效减少锁的密度。

## 语言实现

- (官方实现-不维护)scala：https://github.com/twitter/snowflake/releases/tag/snowflake-2010
- 不同语言github均有各自实现，下面主要介绍PHP实现

# PHP实现

## PHP的劣势

snowflake算法需求在同一毫秒生成的code的序列号自增，由于PHP在语言级别上没有办法让某个对象常驻内存，所以需要借助的其他的方法实现。

### 扩展

[php_snowflake](https://github.com/Sxdd/php_snowflake)，通过扩展实现了让PHP支持全局变量，原理和php的生命周期有关

#### PHP的生命周期

![02-01-01-cgi-lift-cycle](http://static.tongzhao.red/img/20180123204222.png)

#### 多进程SAPI生命周期

![02-01-02-multiprocess-life-cycle](http://static.tongzhao.red/img/20180123204222.png)

通常PHP是编译为apache的一个模块来处理PHP请求。Apache一般会采用多进程模式， Apache启动后会fork出多个子进程，每个进程的内存空间独立，每个子进程都会经过开始和结束环节， 不过每个进程的开始阶段只在进程fork出来以来后进行，在整个进程的生命周期内可能会处理多个请求。 只有在Apache关闭或者进程被结束之后才会进行关闭阶段，在这两个阶段之间会随着每个请求重复请求开始-请求关闭的环节。 如图2.2所示：

#### 多线程的SAPI生命周期

多线程模式和多进程中的某个进程类似，不同的是在整个进程的生命周期
![02-01-013-multithreaded-lift-cycle](http://static.tongzhao.red/img/20180123204222.png)

#### 实现

PHP扩展开发提供了注册module全局变量的功能
http://php.net/manual/zh/internals2.structure.globals.php

小细节：
由于进程关闭后序列会重置，所以算法使用的进程号+序列号保证唯一
提供了service_no可以分布式部署

#### 此扩展的缺点

1. 使用的的32位的字符串，占用空间更大
2. 机器码使用的是进程号，分布式系统下有概率重复（使用不同的service_no则不会）
3. 和snowflake的官方算法有差异，时间戳直接使用当前毫秒数，而非当前时间戳减去固定值（其实使用字符串形式这样更加易读，snowflake官方算法是在为了用更小的位数支持更长的时间）

# 参考

[Twitter-Snowflake，64位自增ID算法详解](http://www.lanindex.com/twitter-snowflake，64位自增id算法详解/)
[PHP生命周期和Zend引擎](http://www.php-internals.com/book/?p=chapt02/02-01-php-life-cycle-and-zend-engine)