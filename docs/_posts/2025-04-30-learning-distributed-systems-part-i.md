---
layout: post
title:  "Learning Distributed Systems - Part I"
date:   2025-04-30 00:10:42 -0400
categories: dist-sys
---

# Part I
## Introduction

I am a self-taught Software Engineer who has dedicated the past three years of my career to both using and later building technologies in the Cloud-Native space, particularly related to Kubernetes. Not having a good college education, I usually tackle the problems I face in this space with the most pragmatic view possible by learning as needed, and it usually works well. 

But a few weeks ago, after tracking down the root cause of a sneaky concurrency bug in a Kubernetes operator, I started to think there might be an upper bound to the kinds of problems I'm able to solve without a structured academic knowledge in the field of Distributed Systems. I've decided to not only set out to learn these topics by myself, but to also document every step of this journey in a series of blog posts.

> The limits of my language mean the limits of my world.

## The resources

The first step to any self-learning endeavor is to select good resources to guide you. With some research around the Web I came up with the following:

 - [Distributed Systems For Fun and Profit by Mikito Takada](https://book.mixu.net/distsys/single-page.html) - A short 70-page book that works as an overview of the field. I read it before doing anything else as I felt it would give me a good map of Distributed Systems to which I can then add more depth.
- [MIT 6.5840](http://nil.csail.mit.edu/6.5840/2024/schedule.html) - The MIT course on distributed systems. I really like it because it has practical labs (with tests!) to be implemented in Go, but its structure seems a little too loose for my taste to be used as a single resource.
- [Designing Data-Intensive Applications by Martin Kleppmann](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321) - A classic, no need for introductions. I will focus mostly on chapters 5-9 which is focused on the theory behind Distributed Systems.
- Papers, papers, papers... - It seems like to really learn about Distributed Systems you have to dig into the foundational papers in the field. Those are referenced extensively in the resources I mentioned previously so the idea is to read them on-demand as I go deeper into certain topics. For starters, "A Note on Distributed Computing" is supposed to be a great introduction.

## Putting the initial knowledge to test

After reading some introductory material and following the first few lectures in *MIT 6.5840* I decided to give the course's second lab a try. I chose to skip the first lab, an implementation of MapReduce as it mostly focused on concepts of concurrency I was already familiar with.

Lab 02 focuses on building a Key/Value server which works over RPC. The server has to tolerate concurrent operations, network failures and use a reasonable amount of memory. MIT provides the basic scaffolding for building the server and also a test suite to make sure your work is correct. You can see my final implementation [here](https://github.com/jbiers/distributed-systems/tree/main/src/kvsrv).

The two first tests were pretty straightforward and only look for a Key/Value store that works for "one client" and "many clients" considering a reliable network connection. I had the main data be stored in a map and included a Mutex in the Server struct to make sure concurrent client accesses would not result in race conditions.

```go
// server.go

type KVServer struct {
	mu sync.Mutex

	data map[string]string
}

func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	reply.Value = kv.data[args.Key]
}

func (kv *KVServer) Put(args *PutAppendArgs, reply *PutAppendReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	kv.data[args.Key] = args.Value

	reply.Value = args.Value
}

func (kv *KVServer) Append(args *PutAppendArgs, reply *PutAppendReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	initialValue := kv.data[args.Key]
	kv.data[args.Key] = initialValue + args.Value

	reply.Value = initialValue
}

```

The next two tests are concerned with fault-tolerance in face of network failures. You need to implement a retry loop in the client. In each RPC, if the server fails to provide a response, the client will keep trying the same operation, as illustrated in the *Get* function below:

```go
// client.go

func (ck *Clerk) Get(key string) string {
	args := GetArgs{
		Key: key,
		ID:  nrand(),
	}
	reply := GetReply{}

	var ok bool
	for {
		ok = ck.server.Call("KVServer.Get", &args, &reply)
		if ok {
			break
		}
	}

	return reply.Value
}
```

This approach works fine for both *Get* and *Put* operations, but *Append* requires some special care as it takes into account the order in which operations happen. In other words, we need to make sure the server is *linearizable*.

My solution to handling *Append* operations when network failures happen is to have each request include a unique identifier. This way, the server can cache all previous *Append* results and, for each incoming request, check if it is a new operation or a retry due to failed communication. In case of a retry, just fetch the value from the cache instead of manipulating data again.

```go
// server.go

func (kv *KVServer) Append(args *PutAppendArgs, reply *PutAppendReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	if value, exists := kv.requestIDs[args.ID]; exists {
		reply.Value = value
		return
	}

	initialValue := kv.data[args.Key]
	kv.data[args.Key] = initialValue + args.Value
	
	// resulting value being cached
	kv.requestIDs[args.ID] = initialValue

	reply.Value = initialValue
}
```

This solution works, but if you pay close attention it's pretty obvious that it introduces a memory leak. The cache map will keep growing and growing with each request. The next few failing tests report this exact problem: too much memory usage. 

The way I solved this problem was inspired in the concept of a *Lamport Clock*, a logical clock that ticks every time a given task is executed. Each RPC now has a unique ID that follows the format `clientID-rpcID` with `rpcID` being a counter that increases with each RPC, the logical clock in question. Given that each client by definition only runs one RPC at a time, we can have the server deleting any entries to the cache that read `clientID-prevRpcID`, being `prevRpcID = rpcID - 1`.

```go
// server.go

type RequestID struct {
	ClientID int64
	RPCCount int
}

func (r *RequestID) GetString() string {
	return fmt.Sprintf("%d-%d", r.ClientID, r.RPCCount)
}

type KVServer struct {
	mu sync.Mutex

	data map[string]string
	requestIDs map[string]string
}

func (kv *KVServer) deleteEntry(r RequestID) {
	delete(kv.requestIDs, r.GetString())
}

func (kv *KVServer) Append(args *PutAppendArgs, reply *PutAppendReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	kv.deleteEntry(RequestID{
		ClientID: args.ID.ClientID,
		RPCCount: args.ID.RPCCount - 1,
	})
	
	if value, exists := kv.requestIDs[args.ID.GetString()]; exists {
		reply.Value = value
		return
	}

	initialValue := kv.data[args.Key]
	kv.data[args.Key] = initialValue + args.Value
	kv.requestIDs[args.ID.GetString()] = initialValue

	reply.Value = initialValue
}
```

With that, we get rid of the memory leak and pass all tests in the lab. 

## Next steps

In the first few days of study I gathered quite a few foundational papers in the field, so next time I'll be diving into some of them and continuing the MIT labs to the implementation of the Raft protocol. Should be fun!

