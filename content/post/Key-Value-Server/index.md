---
title: "MIT 6.5840 Lab2 : Key/Value Server"
date: 2024-02-25 17:06:00+0800
categories:
    - Labs
    - Distributed System
tags:
    - MIT 6.5840
    - RPC
---

## 任务

在一台机器上实现一个 Key/Value Server，要求在有网络请求失败的情况下保证每个操作都恰好执行一次，并且具有线性一致性（linearizability ）。

## 任务1：Key/value server with no network failures

实现没有网络异常情况的 server。直接在正常的 Key/Value 服务的基础上加锁就可以通过对应的前两个测试。

**server.go**

```go
package kvsrv

import (
    "log"
    "sync"
)

const Debug = false

func DPrintf(format string, a ...interface{}) (n int, err error) {
    if Debug {
        log.Printf(format, a...)
    }
    return
}

type KVServer struct {
    mu sync.Mutex
    // Your definitions here.
    data map[string]string
}

func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
    // Your code here.
    kv.mu.Lock()
    defer kv.mu.Unlock()

    // fmt.Printf("Get: key=\"%v\"\n", args.Key)

    reply.Value = kv.data[args.Key]
}

func (kv *KVServer) Put(args *PutAppendArgs, reply *PutAppendReply) {
    // Your code here.
    kv.mu.Lock()
    defer kv.mu.Unlock()

    // fmt.Printf("Put: key=\"%v\" value=\"%v\"\n", args.Key, args.Value)

    reply.Value = kv.data[args.Key]
    kv.data[args.Key] = args.Value
}

func (kv *KVServer) Append(args *PutAppendArgs, reply *PutAppendReply) {
    // Your code here.
    kv.mu.Lock()
    defer kv.mu.Unlock()

    // fmt.Printf("Append: key=\"%v\" value=\"%v\"\n", args.Key, args.Value)

    reply.Value = kv.data[args.Key]
    kv.data[args.Key] = kv.data[args.Key] + args.Value
}

func StartKVServer() *KVServer {
    kv := new(KVServer)

    // You may need initialization code here.
    kv.data = make(map[string]string)

    return kv
}
```

**client.go**

```go
package kvsrv

import (
    "crypto/rand"
    "log"
    "math/big"

    "6.5840/labrpc"
)

type Clerk struct {
    server *labrpc.ClientEnd
    // You will have to modify this struct.
}

func nrand() int64 {
    max := big.NewInt(int64(1) << 62)
    bigx, _ := rand.Int(rand.Reader, max)
    x := bigx.Int64()
    return x
}

func MakeClerk(server *labrpc.ClientEnd) *Clerk {
    ck := new(Clerk)
    ck.server = server
    // You'll have to add code here.
    return ck
}

// fetch the current value for a key.
// returns "" if the key does not exist.
// keeps trying forever in the face of all other errors.
//
// you can send an RPC with code like this:
// ok := ck.server.Call("KVServer.Get", &args, &reply)
//
// the types of args and reply (including whether they are pointers)
// must match the declared types of the RPC handler function's
// arguments. and reply must be passed as a pointer.
func (ck *Clerk) Get(key string) string {
    // fmt.Println("running get")
    // You will have to modify this function.
    args := new(GetArgs)
    reply := new(GetReply)
    args.Key = key
    ok := ck.server.Call("KVServer.Get", args, reply)
    if !ok {
        log.Fatal("Failed to call get.")
        return ""
    }
    return reply.Value
}

// shared by Put and Append.
//
// you can send an RPC with code like this:
// ok := ck.server.Call("KVServer."+op, &args, &reply)
//
// the types of args and reply (including whether they are pointers)
// must match the declared types of the RPC handler function's
// arguments. and reply must be passed as a pointer.
func (ck *Clerk) PutAppend(key string, value string, op string) string {
    // fmt.Println("running PutAppend")
    // You will have to modify this function.
    args := new(PutAppendArgs)
    reply := new(PutAppendReply)
    args.Key = key
    args.Value = value
    var ok bool
    if op == "Put" {
        ok = ck.server.Call("KVServer.Put", args, reply)
        if !ok {
            log.Fatal("Failed to call put.")
            return ""
        }
    } else {
        ok = ck.server.Call("KVServer.Append", args, reply)
        if !ok {
            log.Fatal("Failed to call append.")
            return ""
        }
    }
    return reply.Value
}

func (ck *Clerk) Put(key string, value string) {
    ck.PutAppend(key, value, "Put")
}

// Append value to key's value and return that value
func (ck *Clerk) Append(key string, value string) string {
    return ck.PutAppend(key, value, "Append")
}
```

## 任务2：Key/value server with dropped messages

任务要求修改代码使得 server 能够实现对存在网络异常的情况的正确处理。

client 如果发送 RPC 失败，应该重新发送，而 server 需要保证一个操作只会被执行一次，所以需要具有识别冗余的 RPC 请求的功能。

首先，每个 client 在发送操作请求时生成随机数，用来让 serve 识别此次操作的，用以判断后续收到的是否是冗余请求。对于收到的冗余的 put 请求，只需要直接忽略就行，但对于冗余的 get 和 append 请求，需要返回当时执行请求时对应的返回值。因为冗余请求的产生就是因为网络问题而导致操作执行结束后将返回值返回时 client 没有收到返回信息。因此，需要把对应的当时的返回值返回。

要想在收到冗余请求后返回正确的返回值，需要把操作的返回值存在 client 中，以待冗余请求时返回。但把全部操作的返回值全部存起来太浪费资源了，注意到实验规定一个 client 一次只会发送一个请求到  clerk 中，所以对于每个 client，,可以生成两个随机数来标识当前的 client（两个随机数标识，重复的可能性极小），然后用一个自增的 id 来标识请求。这样，server 只需要存放每个 client 的最后一个请求的返回值就可以了，收到下一个请求证明 client 已经成功接收了上一个请求的返回值，那么就可以将存放的值更新为当前请求的返回值。

但是，这样的方法并不能通过测试，测试结果如下：

![1](kvserv-test-result-1.png)

首先，为什么内存会过大呢？注意到是测试大量 get 请求的测试中内存过大，而测试大量 put 和 append 的测试并没有。这两者有什么区别呢？ get 请求并不会修改服务器上的数据，而 put 和 append 请求会。由此再深入思考，get 请求的返回值另外存放是完全冗余的，因为本身 key 对应的 value 值并没有改变。设想一下，如果没有 put 和 append 请求，对 server 上的每一个 key 发送一个 get 请求，用上面的方法会导致所有 value 全部另外存储，占用了原先两倍的空间，这显然是不合理的。

所以，get 请求不应该像上面那样处理，应该只有在 get 请求的返回值没有确定被 client 成功接收，而对应的 value 又被修改之后存储请求应当返回的原来的 value 值。
