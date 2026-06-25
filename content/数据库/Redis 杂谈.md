---
title: Redis
tags:
  - redis
categories:
  - 数据库
date: 2026-03-18
---
# 基础

## 数据结构

### SDS

Simple dynamic string 即简单动态字符串
#### 二进制安全

重定义字符串，使用结构体属性记录长度

```c
struct __attribute__ ((__packed__)) sdshdr64 { /* packed取消内存对齐 */

    uint64_t len; /* used */

    uint64_t alloc; /* excluding the header and null terminator */

    unsigned char flags; /* 3 lsb of type, 5 unused bits */

    char buf[]; /* 实际存储字符串内容的地方，不依赖'\0'判断结束 */

};
```

#### 动态扩容

```c
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {

    void *sh, *newsh;

    size_t avail = sdsavail(s);

    size_t len, newlen, reqlen;

    char type, oldtype = s[-1] & SDS_TYPE_MASK;

    int hdrlen;

    size_t usable;

  

    /* Return ASAP if there is enough space left. */

    if (avail >= addlen) return s;

  

    len = sdslen(s);

    sh = (char*)s-sdsHdrSize(oldtype);

    reqlen = newlen = (len+addlen);

    assert(newlen > len);   /* Catch size_t overflow */

    if (greedy == 1) {

        if (newlen < SDS_MAX_PREALLOC)

            newlen *= 2;

        else

            newlen += SDS_MAX_PREALLOC;

    }

  

    type = sdsReqType(newlen);

  

    /* Don't use type 5: the user is appending to the string and type 5 is

     * not able to remember empty space, so sdsMakeRoomFor() must be called

     * at every appending operation. */

    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

  

    hdrlen = sdsHdrSize(type);

    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */

    if (oldtype==type) {

        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);

        if (newsh == NULL) return NULL;

        s = (char*)newsh+hdrlen;

    } else {

        /* Since the header size changes, need to move the string forward,

         * and can't use realloc */

        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);

        if (newsh == NULL) return NULL;

        memcpy((char*)newsh+hdrlen, s, len+1);

        s_free(sh);

        s = (char*)newsh+hdrlen;

        s[-1] = type;

        sdssetlen(s, len);

    }

    usable = usable-hdrlen-1;

    if (usable > sdsTypeMaxSize(type))

        usable = sdsTypeMaxSize(type);

    sdssetalloc(s, usable);

    return s;

}
```

### 五种数据结构

```ad-tip
title: This is a tip

This is the content of the admonition tip.
```

#### String

二进制安全，可存储任意二进制数据（可以存储包含`\0`的任意二进制数据）

>**c语言中的传统字符串以"\0"作为结束符，因此一串二进制数据中若包含了"\0"则会在传统字符串中导致存储不完整。**
>
>如不理解这段话，建议去查询c语言类型转换和变量赋值相关的知识，此处不赘述

redis通过sds数据结构解决了二进制安全问题

#### Hash

一个键值对集合， string 类型的 field 和 value 的映射表，适合存对象

#### List

一个字符串列表，支持头尾插入

#### Set

String 的无序集合

#### Zset

String 的有序集合


## 网络模型

## 通信协议

## 内存策略