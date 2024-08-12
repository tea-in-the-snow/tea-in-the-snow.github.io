---
title: "Go RPC"
date: 2024-02-02 11:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - Go
    - RPC
---

RPC(Remote Poresedure Call)是远程方法调用的缩写。Go的RPC库可以实现通过网络或者其他I/O方式远程调用对象的方法。

服务器注册一个对象，让它作为一个以对象类型命名的服务，让这个对象导出的方法可以被远程调用。一个服务器可以注册多个不同类型的对象，但是不能注册同一类型的多个对象。

一个能够被远程调用的方法应该像下面这样：

```go
func (t *T) MethodName(argType T1, replyType *T2) error
```

一个简单的例子：实现简单的kv存储，并（在同一台机器上）通过RPC调用Put和Get方法。

```go
package main

import (
    "fmt"
    "log"
    "net"
    "net/http"
    "net/rpc"
    "sync"
    "time"
)

// RPC struct definition

type PutArgs struct {
    Key string
    Val string
}

type PutReply struct {
    Ok bool
}

type GetArgs struct {
    Key string
}

type GetReply struct {
    Val string
}

type Kv struct {
    data map[string]string
    mu   sync.Mutex
}

// server

func (kv *Kv) Get(args *GetArgs, reply *GetReply) error {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    reply.Val = kv.data[args.Key]
    return nil
}

func (kv *Kv) Put(args *PutArgs, reply *PutReply) error {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    kv.data[args.Key] = args.Val
    reply.Ok = true
    return nil
}

func KvServer() {
    kv := new(Kv)
    kv.data = make(map[string]string)
    rpc.Register(kv)
    rpc.HandleHTTP()
    l, err := net.Listen("tcp", ":1234")
    if err != nil {
        log.Fatal("listen error: ", err)
    }
    http.Serve(l, nil)
}

// client interface

func get(key string) string {
    client, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
    if err != nil {
        log.Fatal("dialing:", err)
    }
    args := GetArgs{key}
    reply := GetReply{}
    e := client.Call("Kv.Get", args, &reply)
    if e != nil {
        log.Fatal("get error", e)
    }
    client.Close()
    return reply.Val
}

func put(key string, val string) bool {
    client, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
    if err != nil {
        log.Fatal("dialing:", err)
    }
    args := PutArgs{key, val}
    reply := PutReply{}
    e := client.Call("Kv.Put", args, &reply)
    if e != nil {
        log.Fatal("put error", e)
    }
    client.Close()
    return reply.Ok
}

// test

func main() {
    go KvServer()
    time.Sleep(time.Second) // wait for server to start
    fmt.Println(put("key1", "value1"))
    fmt.Println(put("key2", "value2"))
    fmt.Println(put("key3", "value3"))
    fmt.Println(get("key1"))
    fmt.Println(get("key2"))
    fmt.Println(get("key3"))
    fmt.Println(get("value1"))
}
```