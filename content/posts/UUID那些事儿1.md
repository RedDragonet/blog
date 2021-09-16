---
title: UUID那些事儿（上）
description:
toc: true
authors: []
tags: [uuid]
categories: [杂谈]
series: []
date: 2017-12-10T20:00:04+08:00
lastmod: 2017-12-10T20:00:04+08:00
featuredVideo:
featuredImage:
draft: false
---



# UUID

UUID 是 通用唯一识别码（Universally Unique Identifier），UUID的目的，是让分散式系统中的所有元素，都能有唯一的辨识资讯，而不需要透过中央控制端来做辨识资讯的指定。
目前最广泛应用的UUID，是微软公司的全局唯一标识符（GUID）
<!--more-->

```
注意
UUID生成算法不保证返回值绝对唯一
170亿分之1的重复概率(数据来源维基百科)
```



## GUID

全局唯一标识符，简称GUID（Globally Unique Identifier），通常表示成32个16进制数字（0－9，A－F）组成的字符串，如：{21EC2020-3AEA-1069-A2DD-08002B30309D}，它实质上是一个128位长的二进制整数。

## UUID的格式

xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
8位 4位 4位 4位 12位
M那个位置，代表版本号，由于UUID的标准实现有5个版本，所以只会是1,2,3,4,5
N那个位置，代表变种(Variant)，一般只会是8,9,a,b
`rfc4122协议10XX四位只能表示成8,9,A(16进制的10),B(16进制的11)`

VariantNCS(已经过时的老古董,NCS=Apollo Network Computing System) 0xxx
VariantRFC4122(本文档) 10XX
VariantMicrosoft(微软) 110x
VariantFuture(未来) 111X

## 各个版本简介

```
http://www.uuid.online/ 可以演示不同版本的UUID
```

UUID目前协议规定的版本有5个，其中1、3、4版本较为常用

### 版本1

```
基于时间戳+MAC地址
```

通过当前时间戳、机器MAC地址生成；
由于在算法中使用了MAC地址，这个版本的UUID可以保证在全球范围的唯一性。
但与此同时，因为它暴露了电脑的MAC地址和生成这个UUID的时间，这就是这个版本UUID被诟病的地方。

### 版本2(大部分不使用)

```
基于时间戳+MAC地址
```

DCE安全的UUID和基于时间的UUID算法相同，但会把时间戳的前4位置换为POSIX的UID或GID。
不过，在UUID的规范里面没有明确地指定，所以基本上所有的UUID实现都不会实现这个版本。

### 版本3

```
基于命名空间的UUID(使用MD5散列化命名空间)
```

由用户指定1个namespace和1个具体的字符串，通过MD5散列，来生成1个UUID；
其中namespace根据协议规定，一般
domain name system, URLs, ISO Object IDs (OIDs), X.500 Distinguished
Names (DNs), and reserved words in a programming language.

### 版本4

```
基于随机数的UUID
```

最简单常用的一种

### 版本5

```
基于命名空间的UUID(使用SHA1散列化命名空间)
```

## 代码示例

```Python
import uuid
print "v1=",uuid.uuid1()
print "v3=",uuid.uuid3(uuid.NAMESPACE_DNS, "myString")
print "v4=",uuid.uuid4()
print "v5=",uuid.uuid5(uuid.NAMESPACE_DNS, "myString")
//java只实现了v3和v4
import java.util.UUID;

public class code
{
    public static void main(String[] args)
    {
        System.out.println("v3="+UUID.nameUUIDFromBytes("myString".getBytes()).toString());
        System.out.println("v4="+UUID.randomUUID());
    }
}
```

## UUID和各个编程语言

```
注:PHP内置函数uniqid并不是uuid协议的实现
```

- PHP：https://github.com/ramsey/uuid
- Java：http://docs.oracle.com/javase/7/docs/api/java/util/UUID.html
- Golang：https://godoc.org/github.com/satori/go.uuid
- Android：http://developer.android.com/reference/java/util/UUID.html
- IOS：https://developer.apple.com/documentation/foundation/uuid
- nodejs - https://www.npmjs.com/package/uuid
- 微软：http://msdn.microsoft.com/en-us/library/system.guid(v=vs.110).aspx
- Linux：http://en.wikipedia.org/wiki/Util-linux
- MySQL:http://dev.mysql.com/doc/refman/5.1/en/miscellaneous-functions.html#function_uuid

## PHP uniqid

```
//prefix 自定义字符串，如服务器的mac地址，ip等，保证多台机器同一微秒生成的值不重复
//more_entropy 会在返回的字符串结尾增加额外的熵,增加随机性
string uniqid ([ string $prefix = "" [, bool $more_entropy = false ]] )
```

获取一个带前缀、基于当前时间微秒数的唯一ID。

```php
//代码示例
echo uniqid();//5a2cf3024ee36
//部分PHP源码
//uniqid.c
gettimeofday((struct timeval *) &tv, (struct timezone *) NULL);
sec = (int) tv.tv_sec;
usec = (int) (tv.tv_usec % 0x100000);
    
if (more_entropy) {
    spprintf(&uniqid, 0, "%s%08x%05x%.8F", prefix, sec, usec, php_combined_lcg(TSRMLS_C) * 10);
} else {
    spprintf(&uniqid, 0, "%s%08x%05x", prefix, sec, usec);
}
```

## 参考

[关于UUID的二三事](http://www.jianshu.com/p/d77f3ef0868a)
[rfc4122](https://tools.ietf.org/html/rfc4122)
[Universally unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier#Variants_and_versions)
[通用唯一识别码](https://zh.wikipedia.org/wiki/通用唯一识别码)