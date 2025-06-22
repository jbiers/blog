---
layout: post
title:  "Learning Distributed Systems - Part II - Single Leader Data Replication"
date:   2025-06-22 00:10:42 -0400
categories: dist-sys
---

# Part II - Single Leader Data Replication
## Introduction

In this post we're going to explore concepts around data replication in distributed systems.
For now, we are dealing only with single-leader replication scenarios. Multi-leader replication necessarily involves handling write conflicts which is a huge topic in itself and opens up the umbrella of consensus algorithms. Those topics will be covered in the following posts.

> I highly recommend you read the [Google File System](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf) paper which presents the architecture of a system developed by Google in 2003 designed to be a distributed file system. It was implemented following the single-leader pattern and is a great deeper dive into the concepts presented in this post.

## Why replicate data?

Several reasons lead us to the need of replicating data across multiple replicas, otherwise no one would bother dealing with the added complexity brought by such replications. The main reasons are:

- Reduce latency by increasing geographical proximity to users. Remember that requests are limited by the speed of light, so traveling shorter distances is always a plus. 
- Increase availability. If a replica fails, the client's request can just be routed to another one.
- Scaling the system beyond the power of a single machine. Horizontal scaling (adding more machines to a cluster) is usually cheaper than vertical scaling (improving each machine's resources), especially in systems of large proportions.

## Leaders and Followers

In this post we assume we are dealing with data that is constantly changing, that is, can be altered via write requests. Replication of static data is a lot more trivial and will not be covered here.

The simplest way to handle these scenarios is to divide machines between a Leader and Followers. By definition, a Leader is a replica that processes writes to the data. Followers, on the other hand, can only process reads. The Master is also responsible for keeping track of each Follower's status, to understand whether they are alive of dead.

Focusing all writes to a single replica is a great strategy in a lot of cases as it prevents systems from having to deal with write conflicts (multiple clients writing to the same data at the same time), and it fits well with the pattern seen in multiple web applications of reads being a lot more common than writes. To illustrate that, think of how many posts you see versus how many posts you create on Instagram or any other social media platform.

## Replicating changes to the data

Let's now walk through the process of replicating a write request in a single-leader system in detail.

First, the Leader receives the write request from a client. The immediate action is to persist the change to its local disk. This can mean actually performing the data write, or, as is the case in a lot of systems like GFS, it will only append a log entry to a log file and only then handle the data write. This technique is known as write-ahead logging and is very useful to improve write performance as a log append is a lot faster than modifying the actual data.

Now, there are two alternatives. The Leader can either send the response of the write request to the client instantly after writing the log without waiting for data to be replicated to the Followers (known as the *asynchronous approach*) or it can wait until data has been fully replicated to only then send the response back (known as the *synchronous approach*).

They both have their pros and cons. *Asynchronous replication* is a lot faster, but relies on the idea of *eventual consistency* due to replication lag. As an example, if a user issues a request to add a comment to a post and then wants to see the comment they added (a very common pattern known as *read-your-writes*), the read request might be directed to a Follower that still hasn't caught up to the Leader, giving the user the false impression that their comment doesn't exist.

*Synchronous replication*, on the other hand, guarantees such a scenario won't happen. The downsides are the obvious decreased performance and, even worse, the possibility of stalling the Leader if one of the Followers fails to respond to the data replication in case of node failure or a network partition.

![single-leader-replication]({{ site.baseurl }}/assets/images/single-leader.png)

## What happens if a replica fails?

In distributed systems we always have to think of worst-case scenarios as they are bound to happen when using multiple machine over a long period of time.

### Follower failure.

The first example and easiest case to deal with is that of a Follower either dying or for some other reason not responding. The Leader will detect a node is down if it doesn't respond to the Heartbeat request for a given period of time, and will then stop replicating writes to it. When the node comes back online after being restarted, it will ask the Leader to send it all logs that were processed since it went down. After processing the logs, the Follower is back to being a live node ready to receive reads.

### Leader failure.

Leader failure is more problematic. Since all writes go through the Leader, its failure renders the system unable to process write requests. Handling Leader failure is called **failover** and is one of the trickiest areas of distributed systems design.

There are two types of failover: manual and automatic.

Manual failover is handled by human operators. An engineer inspects the system, chooses the most up-to-date Follower, and promotes it to Leader. This process is typically safer but takes longer, which might be unacceptable for systems requiring high availability.

Automatic failover tries to detect the failure and promote a new Leader without human intervention. While faster, it comes with a set of challenges.

One major challenge is leader election: how do you choose a new Leader reliably, especially under uncertain network conditions? For example, imagine a situation where the old Leader is still alive but just temporarily unreachable due to a network partition. If another node is promoted too early, you end up with split brain — two nodes acting as Leader simultaneously, leading to conflicting writes and possible data corruption. To avoid this, distributed systems often use consensus algorithms like Paxos or Raft to ensure a safe, coordinated Leader election. These algorithms guarantee that even in the presence of failures or network delays, only one node will be considered the legitimate Leader.

## Next steps

The next post will cover cases of multi-leader replication and touch on the ideas of handling write conflicts and consensus algorithms.
