## lab2 预备知识

## raft locking advice

### 英文原文

If you are wondering how to use locks in the 6.824 Raft labs, here are
some rules and ways of thinking that might be helpful.

Rule 1: Whenever you have data that more than one goroutine uses, and
at least one goroutine might modify the data, the goroutines should
use locks to prevent simultaneous use of the data. The Go race
detector is pretty good at detecting violations of this rule (though
it won't help with any of the rules below).

Rule 2: Whenever code makes a sequence of modifications to shared
data, and other goroutines might malfunction if they looked at the
data midway through the sequence, you should use a lock around the
whole sequence.

```go
// An example:

rf.mu.Lock()
rf.currentTerm += 1
rf.state = Candidate
rf.mu.Unlock()
```

It would be a mistake for another goroutine to see either of these
updates alone (i.e. the old state with the new term, or the new term
with the old state). So we need to hold the lock continuously over the
whole sequence of updates. All other code that uses rf.currentTerm or
rf.state must also hold the lock, in order to ensure exclusive access
for all uses.

The code between Lock() and Unlock() is often called a "critical
section." The locking rules a programmer chooses (e.g. "a goroutine
must hold rf.mu when using rf.currentTerm or rf.state") are often
called a "locking protocol".

Rule 3: Whenever code does a sequence of reads of shared data (or
reads and writes), and would malfunction if another goroutine modified
the data midway through the sequence, you should use a lock around the
whole sequence.

```go
// An example that could occur in a Raft RPC handler:

rf.mu.Lock()
if args.Term > rf.currentTerm {
rf.currentTerm = args.Term
}
rf.mu.Unlock()
```

This code needs to hold the lock continuously for the whole sequence.
Raft requires that currentTerm only increases, and never decreases.
Another RPC handler could be executing in a separate goroutine; if it
were allowed to modify rf.currentTerm between the if statement and the
update to rf.currentTerm, this code might end up decreasing
rf.currentTerm. Hence the lock must be held continuously over the
whole sequence. In addition, every other use of currentTerm must hold
the lock, to ensure that no other goroutine modifies currentTerm
during our critical section.

Real Raft code would need to use longer critical sections than these
examples; for example, a Raft RPC handler should probably hold the
lock for the entire handler.

Rule 4: It's usually a bad idea to hold a lock while doing anything
that might wait: reading a Go channel, sending on a channel, waiting
for a timer, calling time.Sleep(), or sending an RPC (and waiting for the
reply). One reason is that you probably want other goroutines to make
progress during the wait. Another reason is deadlock avoidance. Imagine
two peers sending each other RPCs while holding locks; both RPC
handlers need the receiving peer's lock; neither RPC handler can ever
complete because it needs the lock held by the waiting RPC call.

Code that waits should first release locks. If that's not convenient,
sometimes it's useful to create a separate goroutine to do the wait.

Rule 5: Be careful about assumptions across a drop and re-acquire of a
lock. One place this can arise is when avoiding waiting with locks
held. For example, this code to send vote RPCs is incorrect:

```go
rf.mu.Lock()
rf.currentTerm += 1
rf.state = Candidate
for <each peer> {
  go func() {
    rf.mu.Lock()
    args.Term = rf.currentTerm
    rf.mu.Unlock()
    Call("Raft.RequestVote", &args, ...)
    // handle the reply...
  } ()
}
rf.mu.Unlock()
```

The code sends each RPC in a separate goroutine. It's incorrect
because args.Term may not be the same as the rf.currentTerm at which
the surrounding code decided to become a Candidate. Lots of time may
pass between when the surrounding code creates the goroutine and when
the goroutine reads rf.currentTerm; for example, multiple terms may
come and go, and the peer may no longer be a candidate. One way to fix
this is for the created goroutine to use a copy of rf.currentTerm made
while the outer code holds the lock. Similarly, reply-handling code
after the Call() must re-check all relevant assumptions after
re-acquiring the lock; for example, it should check that
rf.currentTerm hasn't changed since the decision to become a
candidate.

It can be difficult to interpret and apply these rules. Perhaps most
puzzling is the notion in Rules 2 and 3 of code sequences that
shouldn't be interleaved with other goroutines' reads or writes. How
can one recognize such sequences? How should one decide where a
sequence ought to start and end?

One approach is to start with code that has no locks, and think
carefully about where one needs to add locks to attain correctness.
This approach can be difficult since it requires reasoning about the
correctness of concurrent code.

A more pragmatic approach starts with the observation that if there
were no concurrency (no simultaneously executing goroutines), you
would not need locks at all. But you have concurrency forced on you
when the RPC system creates goroutines to execute RPC handlers, and
because you need to send RPCs in separate goroutines to avoid waiting.
You can effectively eliminate this concurrency by identifying all
places where goroutines start (RPC handlers, background goroutines you
create in Make(), &c), acquiring the lock at the very start of each
goroutine, and only releasing the lock when that goroutine has
completely finished and returns. This locking protocol ensures that
nothing significant ever executes in parallel; the locks ensure that
each goroutine executes to completion before any other goroutine is
allowed to start. With no parallel execution, it's hard to violate
Rules 1, 2, 3, or 5. If each goroutine's code is correct in isolation
(when executed alone, with no concurrent goroutines), it's likely to
still be correct when you use locks to suppress concurrency. So you
can avoid explicit reasoning about correctness, or explicitly
identifying critical sections.

However, Rule 4 is likely to be a problem. So the next step is to find
places where the code waits, and to add lock releases and re-acquires
(and/or goroutine creation) as needed, being careful to re-establish
assumptions after each re-acquire. You may find this process easier to
get right than directly identifying sequences that must be locked for
correctness.

(As an aside, what this approach sacrifices is any opportunity for
better performance via parallel execution on multiple cores: your code
is likely to hold locks when it doesn't need to, and may thus
unnecessarily prohibit parallel execution of goroutines. On the other
hand, there is not much opportunity for CPU parallelism within a
single Raft peer.)

### 中文翻译

规则1：每当你有多个goroutine使用的数据，并且至少有一个goroutine可能修改数据时，这些goroutine应该使用锁来防止数据的同时使用。Go的竞态检测器在检测违反此规则方面非常有效（尽管它对下面的规则没有帮助）。

规则2：每当代码对共享数据进行一系列修改，并且其他goroutine在这个序列的中途查看数据可能会出现故障时，你应该在整个序列周围使用锁。

```go
// An example:

rf.mu.Lock()
rf.currentTerm += 1
rf.state = Candidate
rf.mu.Unlock()
```

如果另一个goroutine只看到其中一个更新（即旧状态与新任期，或新任期与旧状态），那将是一个错误。因此，我们需要在整个更新序列中持续持有锁。所有使用rf.currentTerm或rf.state的其他代码也必须持有锁，以确保所有使用的独占访问。

在Lock()和Unlock()之间的代码通常被称为“临界区”。程序员选择的锁定规则（例如，“使用rf.currentTerm或rf.state时，xn--goroutinerf-km8q8qw94r4ek1hqek2r.mu”）通常被称为“锁定协议”。

规则3：每当代码对共享数据进行一系列读取（或读取和写入），并且如果另一个goroutine在序列的中途修改数据会导致故障时，你应该在整个序列周围使用锁。

```go
// An example that could occur in a Raft RPC handler:

rf.mu.Lock()
if args.Term > rf.currentTerm {
rf.currentTerm = args.Term
}
rf.mu.Unlock()
```

这段代码需要在整个序列中持续持有锁。Raft要求currentTerm只能增加，不能减少。另一个RPC处理程序可能在一个单独的goroutine中执行；如果允许它在if语句和对rf.currentTerm的更新之间修改rf.currentTerm，那么这段代码可能会导致rf.currentTerm减小。因此，锁必须在整个序列中持续持有。此外，对currentTerm的每个其他使用都必须持有锁，以确保在我们的临界区中没有其他goroutine修改currentTerm。

实际的Raft代码需要使用比这些示例更长的临界区；例如，Raft的RPC处理程序可能应该在整个处理程序中持有锁。

规则4：==在执行可能会等待的操作时，通常不要持有锁==：==读取Go通道、在通道上发送、等待计时器、调用time.Sleep()或发送RPC（并等待回复）==。一个原因是在等待期间，你可能希望其他goroutine取得进展。另一个原因是避免死锁。想象一下，两个对等方在持有锁的同时互相发送RPC；两个RPC处理程序都需要接收方的锁；由于它需要等待的RPC调用持有锁，因此两个RPC处理程序都无法完成。

等待的代码应该首先释放锁。如果这样做不方便，有时创建一个单独的goroutine来进行等待是有用的。

规则5：在放弃和重新获取锁之间，要小心对假设的处理。一个可能出现这种情况的地方是在持有锁的情况下避免等待。例如，发送投票RPC的以下代码是不正确的：

```go
rf.mu.Lock()
rf.currentTerm += 1
rf.state = Candidate
for <each peer> {
  go func() {
    rf.mu.Lock() // 在创建 goroutine 的时候，可能 currentTerm产生了变化，所以需要在外部获取副本
    args.Term = rf.currentTerm
    rf.mu.Unlock()
    Call("Raft.RequestVote", &args, ...)
    // handle the reply...
  } ()
}
rf.mu.Unlock()
```

这段代码将每个RPC都在一个单独的goroutine中发送。然而，这是==错误的==，因为args.Term可能与包围代码决定成为候选人的rf.currentTerm不同。**在包围代码创建goroutine和goroutine读取rf.currentTerm之间可能会经过很长时间**。例如，可能会有多个任期来来去去，而该对等体可能不再是候选人。修复这个问题的一种方法是创建的goroutine在外部代码持有锁时使用rf.currentTerm的副本。类似地，在Call()之后的回复处理代码在重新获取锁之后==必须重新检查所有相关的假设==；例如，它应该检查rf.currentTerm在决定成为候选人后是否发生了变化。

解释和应用这些规则可能会有一定的难度。规则2和3中最令人困惑的概念是代码序列不应与其他goroutine的读取或写入交错。如何识别这样的序列？如何确定序列应该从何处开始和结束？

一种方法是从没有锁的代码开始，并仔细考虑在哪些地方需要添加锁以确保正确性。这种方法可能很困难，因为它需要对并发代码的正确性进行推理。

一个更实用的方法是从这样一个观察开始：如果没有并发（没有同时执行的goroutine），你根本不需要锁。但是，当RPC系统创建goroutine来执行RPC处理程序并且因为你需要在单独的goroutine中发送RPC来避免等待时，你被迫面对并发。通过**识别所有启动goroutine的地方**（RPC处理程序，在Make()中创建的后台goroutine等），在==每个goroutine的开始处获取锁==，并且只有在==该goroutine完全完成并返回后才释放锁==，你可以有效地消除这种并发。这种锁定协议确保没有重要的东西会并行执行；锁定确保每个goroutine在任何其他goroutine被允许启动之前都会完整地执行完毕。没有并行执行，很难违反规则1、2、3或5。如果每个goroutine的代码在隔离的情况下是正确的（在没有并发的goroutine情况下单独执行时），当你使用锁来抑制并发时，它可能仍然是正确的。因此，你可以避免明确推理正确性，或明确识别关键部分。

然而，规则4可能会成为一个问题。所以下一步是找出代码等待的地方，并根据需要添加锁释放和重新获取（和/或goroutine创建），在每次重新获取后小心重新建立假设。你可能会发现这个过程比直接识别必须为了正确性而加锁的序列更容易做到。

（顺便说一句，这种方法牺牲的是通过在多个核心上并行执行来提高性能的机会：你的代码可能会在不需要的时候持有锁，因此可能不必要地阻止了goroutine的并行执行。另一方面，在单个Raft对等体内，CPU并行性的机会并不多。）

## raft structure advice

### 英文原文

A Raft instance has to deal with the arrival of external events
(Start() calls, AppendEntries and RequestVote RPCs, and RPC replies),
and it has to execute periodic tasks (elections and heart-beats).
There are many ways to structure your Raft code to manage these
activities; this document outlines a few ideas.

Each Raft instance has a bunch of state (the log, the current index,
&c) which must be updated in response to events arising in concurrent
goroutines. The Go documentation points out that the goroutines can
perform the updates directly using shared data structures and locks,
or by passing messages on channels. Experience suggests that for Raft
it is most straightforward to use shared data and locks.

A Raft instance has two time-driven activities: the leader must send
heart-beats, and others must start an election if too much time has
passed since hearing from the leader. It's probably best to drive each
of these activities with a dedicated long-running goroutine, rather
than combining multiple activities into a single goroutine.

The management of the election timeout is a common source of
headaches. Perhaps the simplest plan is to maintain a variable in the
Raft struct containing the last time at which the peer heard from the
leader, and to have the election timeout goroutine periodically check
to see whether the time since then is greater than the timeout period.
It's easiest to use time.Sleep() with a small constant argument to
drive the periodic checks. Don't use time.Ticker and time.Timer;
they are tricky to use correctly.

You'll want to have a separate long-running goroutine that sends
committed log entries in order on the applyCh. It must be separate,
since sending on the applyCh can block; and it must be a single
goroutine, since otherwise it may be hard to ensure that you send log
entries in log order. The code that advances commitIndex will need to
kick the apply goroutine; it's probably easiest to use a condition
variable (Go's sync.Cond) for this.

Each RPC should probably be sent (and its reply processed) in its own
goroutine, for two reasons: so that unreachable peers don't delay the
collection of a majority of replies, and so that the heartbeat and
election timers can continue to tick at all times. It's easiest to do
the RPC reply processing in the same goroutine, rather than sending
reply information over a channel.

Keep in mind that the network can delay RPCs and RPC replies, and when
you send concurrent RPCs, the network can re-order requests and
replies. Figure 2 is pretty good about pointing out places where RPC
handlers have to be careful about this (e.g. an RPC handler should
ignore RPCs with old terms). Figure 2 is not always explicit about RPC
reply processing. The leader has to be careful when processing
replies; it must check that the term hasn't changed since sending the
RPC, and must account for the possibility that replies from concurrent
RPCs to the same follower have changed the leader's state (e.g.
nextIndex).

### 中文翻译

一个Raft实例==必须处理外部事件的到达==（**Start()调用，AppendEntries和RequestVote RPC，以及RPC回复**），并且必须==执行定期任务==（选举和心跳）。有许多方法可以组织你的Raft代码来管理这些活动；本文档概述了一些想法。

每个Raft实例都有一堆状态（日志，当前索引等），必须根据**并发goroutine中出现的事件进行更新**。Go文档指出，goroutine可以**直接使用共享数据结构和锁来执行更新**，也可以通过通道传递消息。经验表明，对于Raft来说，使用==共享数据和锁最直接==。

一个Raft实例有两个时间驱动的活动：领导者必须发送心跳，其他节点必须在听到领导者的消息时间过长后启动选举。最好==为每个活动驱动一个专用的长时间运行的goroutine==（**发送心跳和选举改成单独的goroutine**），而不是将多个活动合并到一个单独的goroutine中。

==选举超时的管理==是一个常见的头疼问题。也许最简单的方法是==在Raft结构体中维护一个变量==，其中==包含节点上次从领导者收到消息的时间，并且选举超时的goroutine定期检查从那时起的时间是否超过超时时间==。使用time.Sleep()和一个小的常量参数来驱动定期检查是最简单的。不要使用time.Ticker和time.Timer；==它们很难正确使用==。

你需要有一个单独的长时间运行的goroutine，按顺序在applyCh上发送已提交的日志条目。它必须是独立的，因为在applyCh上==发送可能会阻塞==；而且它==必须是一个单独的goroutine==，否则很难确保==按日志顺序==发送日志条目。推进commitIndex的代码将需要唤醒apply goroutine；最容易使用条件变量（Go的sync.Cond）来实现这一点。

每个RPC应该在它自己的goroutine中发送（并处理其回复），有两个原因：一是为了避免无法到达的对等体==延迟收集大多数回复==，二是为了==保证心跳和选举计时器能够始终正常工作==。最简单的做法是在==同一个goroutine中处理RPC回复==，而不是通过通道发送回复信息。

请记住，网络可能会延迟RPC和RPC回复，并且当你发送并发的RPC时，网络可能会==重新排序请求和回复==。图2在指出RPC处理程序必须注意这一点的地方做得很好（例如，RPC处理程序应该==忽略具有旧任期的RPC==）。图2并不总是明确说明RPC回复处理。领导者在处理回复时必须小心；它必须检查自==从发送RPC以来任期是否发生了变化==，并且必须考虑==同时发送给同一追随者的并发RPC的回复可能会改变领导者的状态==（例如，nextIndex）。

## raft 指南

### 英文原文

For the past few months, I have been a Teaching Assistant for MIT’s [6.824 Distributed Systems](https://pdos.csail.mit.edu/6.824/) class. The class has traditionally had a number of labs building on the Paxos consensus algorithm, but this year, we decided to make the move to [Raft](https://raft.github.io/). Raft was “designed to be easy to understand”, and our hope was that the change might make the students’ lives easier.

This post, and the accompanying [Instructors’ Guide to Raft](https://thesquareplanet.com/blog/instructors-guide-to-raft/) post, chronicles our journey with Raft, and will hopefully be useful to implementers of the Raft protocol and students trying to get a better understanding of Raft’s internals. If you are looking for a Paxos vs Raft comparison, or for a more pedagogical analysis of Raft, you should go read the Instructors’ Guide. The bottom of this post contains a list of questions commonly asked by 6.824 students, as well as answers to those questions. If you run into an issue that is not listed in the main content of this post, check out the [Q&A](https://thesquareplanet.com/blog/raft-qa/). The post is quite long, but all the points it makes are real problems that a lot of 6.824 students (and TAs) ran into. It is a worthwhile read.

### [Background](https://thesquareplanet.com/blog/students-guide-to-raft/#background)

Before we dive into Raft, some context may be useful. 6.824 used to have a set of [Paxos-based labs](http://nil.csail.mit.edu/6.824/2015/labs/lab-3.html) that were built in [Go](https://golang.org/); Go was chosen both because it is easy to learn for students, and because is pretty well-suited for writing concurrent, distributed applications (goroutines come in particularly handy). Over the course of four labs, students build a fault-tolerant, sharded key-value store. The first lab had them build a consensus-based log library, the second added a key value store on top of that, and the third sharded the key space among multiple fault-tolerant clusters, with a fault-tolerant shard master handling configuration changes. We also had a fourth lab in which the students had to handle the failure and recovery of machines, both with and without their disks intact. This lab was available as a default final project for students.

This year, we decided to rewrite all these labs using Raft. The first three labs were all the same, but the fourth lab was dropped as persistence and failure recovery is already built into Raft. This article will mainly discuss our experiences with the first lab, as it is the one most directly related to Raft, though I will also touch on building applications on top of Raft (as in the second lab).

Raft, for those of you who are just getting to know it, is best described by the text on the protocol’s [web site](https://raft.github.io/):

> Raft is a consensus algorithm that is designed to be easy to understand. It’s equivalent to Paxos in fault-tolerance and performance. The difference is that it’s decomposed into relatively independent subproblems, and it cleanly addresses all major pieces needed for practical systems. We hope Raft will make consensus available to a wider audience, and that this wider audience will be able to develop a variety of higher quality consensus-based systems than are available today.

Visualizations like [this one](http://thesecretlivesofdata.com/raft/) give a good overview of the principal components of the protocol, and the paper gives good intuition for why the various pieces are needed. If you haven’t already read the [extended Raft paper](https://raft.github.io/raft.pdf), you should go read that before continuing this article, as I will assume a decent familiarity with Raft.

As with all distributed consensus protocols, the devil is very much in the details. In the steady state where there are no failures, Raft’s behavior is easy to understand, and can be explained in an intuitive manner. For example, it is simple to see from the visualizations that, assuming no failures, a leader will eventually be elected, and eventually all operations sent to the leader will be applied by the followers in the right order. However, when delayed messages, network partitions, and failed servers are introduced, each and every if, but, and and, become crucial. In particular, there are a number of bugs that we see repeated over and over again, simply due to misunderstandings or oversights when reading the paper. This problem is not unique to Raft, and is one that comes up in all complex distributed systems that provide correctness.

### [Implementing Raft](https://thesquareplanet.com/blog/students-guide-to-raft/#implementing-raft)

The ultimate guide to Raft is in Figure 2 of the Raft paper. This figure specifies the behavior of every RPC exchanged between Raft servers, gives various invariants that servers must maintain, and specifies when certain actions should occur. We will be talking about Figure 2 *a lot* in the rest of this article. It needs to be followed *to the letter*.

Figure 2 defines what every server should do, in ever state, for every incoming RPC, as well as when certain other things should happen (such as when it is safe to apply an entry in the log). At first, you might be tempted to treat Figure 2 as sort of an informal guide; you read it once, and then start coding up an implementation that follows roughly what it says to do. Doing this, you will quickly get up and running with a mostly working Raft implementation. And then the problems start.

In fact, Figure 2 is extremely precise, and every single statement it makes should be treated, in specification terms, as **MUST**, not as **SHOULD**. For example, you might reasonably reset a peer’s election timer whenever you receive an or RPC, as both indicate that some other peer either thinks it’s the leader, or is trying to become the leader. Intuitively, this means that we shouldn’t be interfering. However, if you read Figure 2 carefully, it says:`AppendEntries``RequestVote`

> If election timeout elapses without receiving RPC *from current leader* or *granting* vote to candidate: convert to candidate.`AppendEntries`

The distinction turns out to matter a lot, as the former implementation can result in significantly reduced liveness in certain situations.

#### [The importance of details](https://thesquareplanet.com/blog/students-guide-to-raft/#the-importance-of-details)

To make the discussion more concrete, let us consider an example that tripped up a number of 6.824 students. The Raft paper mentions *heartbeat RPCs* in a number of places. Specifically, a leader will occasionally (at least once per heartbeat interval) send out an RPC to all peers to prevent them from starting a new election. If the leader has no new entries to send to a particular peer, the RPC contains no entries, and is considered a heartbeat.`AppendEntries``AppendEntries`

Many of our students assumed that heartbeats were somehow “special”; that when a peer receives a heartbeat, it should treat it differently from a non-heartbeat RPC. In particular, many would simply reset their election timer when they received a heartbeat, and then return success, without performing any of the checks specified in Figure 2. This is *extremely dangerous*. By accepting the RPC, the follower is implicitly telling the leader that their log matches the leader’s log up to and including the included in the arguments. Upon receiving the reply, the leader might then decide (incorrectly) that some entry has been replicated to a majority of servers, and start committing it.`AppendEntries``prevLogIndex``AppendEntries`

Another issue many had (often immediately after fixing the issue above), was that, upon receiving a heartbeat, they would truncate the follower’s log following , and then append any entries included in the arguments. This is *also* not correct. We can once again turn to Figure 2:`prevLogIndex``AppendEntries`

> *If* an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it.

The *if* here is crucial. If the follower has all the entries the leader sent, the follower **MUST NOT** truncate its log. Any elements *following* the entries sent by the leader **MUST** be kept. This is because we could be receiving an outdated RPC from the leader, and truncating the log would mean “taking back” entries that we may have already told the leader that we have in our log.`AppendEntries`

### [Debugging Raft](https://thesquareplanet.com/blog/students-guide-to-raft/#debugging-raft)

Inevitably, the first iteration of your Raft implementation will be buggy. So will the second. And third. And fourth. In general, each one will be less buggy than the previous one, and, from experience, most of your bugs will be a result of not faithfully following Figure 2.

When debugging, Raft, there are generally four main sources of bugs: livelocks, incorrect or incomplete RPC handlers, failure to follow The Rules, and term confusion. Deadlocks are also a common problem, but they can generally be debugged by logging all your locks and unlocks, and figuring out which locks you are taking, but not releasing. Let us consider each of these in turn:

#### [Livelocks](https://thesquareplanet.com/blog/students-guide-to-raft/#livelocks)

When your system livelocks, every node in your system is doing something, but collectively your nodes are in such a state that no progress is being made. This can happen fairly easily in Raft, especially if you do not follow Figure 2 religiously. One livelock scenario comes up especially often; no leader is being elected, or once a leader is elected, some other node starts an election, forcing the recently elected leader to abdicate immediately.

There are many reasons why this scenario may come up, but there is a handful of mistakes that we have seen numerous students make:

- Make sure you reset your election timer *exactly* when Figure 2 says you should. Specifically, you should *only* restart your election timer if a) you get an RPC from the *current* leader (i.e., if the term in the arguments is outdated, you should *not* reset your timer); b) you are starting an election; or c) you *grant* a vote to another peer.`AppendEntries``AppendEntries`

  This last case is especially important in unreliable networks where it is likely that followers have different logs; in those situations, you will often end up with only a small number of servers that a majority of servers are willing to vote for. If you reset the election timer whenever someone asks you to vote for them, this makes it equally likely for a server with an outdated log to step forward as for a server with a longer log.

  In fact, because there are so few servers with sufficiently up-to-date logs, those servers are quite unlikely to be able to hold an election in sufficient peace to be elected. If you follow the rule from Figure 2, the servers with the more up-to-date logs won’t be interrupted by outdated servers’ elections, and so are more likely to complete the election and become the leader.

- Follow Figure 2’s directions as to when you should start an election. In particular, note that if you are a candidate (i.e., you are currently running an election), but the election timer fires, you should start *another* election. This is important to avoid the system stalling due to delayed or dropped RPCs.

- Ensure that you follow the second rule in “Rules for Servers” *before* handling an incoming RPC. The second rule states:

  > If RPC request or response contains term : set , convert to follower (§5.1)`T > currentTerm``currentTerm = T`

  For example, if you have already voted in the current term, and an incoming RPC has a higher term that you, you should *first* step down and adopt their term (thereby resetting ), and *then* handle the RPC, which will result in you granting the vote!`RequestVote``votedFor`

#### [Incorrect RPC handlers](https://thesquareplanet.com/blog/students-guide-to-raft/#incorrect-rpc-handlers)

Even though Figure 2 spells out exactly what each RPC handler should do, some subtleties are still easy to miss. Here are a handful that we kept seeing over and over again, and that you should keep an eye out for in your implementation:

- If a step says “reply false”, this means you should *reply immediately*, and not perform any of the subsequent steps.
- If you get an RPC with a that points beyond the end of your log, you should handle it the same as if you did have that entry but the term did not match (i.e., reply false).`AppendEntries``prevLogIndex`
- Check 2 for the RPC handler should be executed *even if the leader didn’t send any entries*.`AppendEntries`
- The in the final step (#5) of is *necessary*, and it needs to be computed with the index of the last *new* entry. It is *not* sufficient to simply have the function that applies things from your log between and stop when it reaches the end of your log. This is because you may have entries in your log that differ from the leader’s log *after* the entries that the leader sent you (which all match the ones in your log). Because #3 dictates that you only truncate your log *if* you have conflicting entries, those won’t be removed, and if is beyond the entries the leader sent you, you may apply incorrect entries.`min``AppendEntries``lastApplied``commitIndex``leaderCommit`
- It is important to implement the “up-to-date log” check *exactly* as described in section 5.4. No cheating and just checking the length!

#### [Failure to follow The Rules](https://thesquareplanet.com/blog/students-guide-to-raft/#failure-to-follow-the-rules)

While the Raft paper is very explicit about how to implement each RPC handler, it also leaves the implementation of a number of rules and invariants unspecified. These are listed in the “Rules for Servers” block on the right hand side of Figure 2. While some of them are fairly self-explanatory, the are also some that require designing your application very carefully so that it does not violate The Rules:

- If *at any point* during execution, you should apply a particular log entry. It is not crucial that you do it straight away (for example, in the RPC handler), but it *is* important that you ensure that this application is only done by one entity. Specifically, you will need to either have a dedicated “applier”, or to lock around these applies, so that some other routine doesn’t also detect that entries need to be applied and also tries to apply.`commitIndex > lastApplied``AppendEntries`
- Make sure that you check for either periodically, or after is updated (i.e., after is updated). For example, if you check at the same time as sending out to peers, you may have to wait until the *next* entry is appended to the log before applying the entry you just sent out and got acknowledged.`commitIndex > lastApplied``commitIndex``matchIndex``commitIndex``AppendEntries`
- If a leader sends out an RPC, and it is rejected, but *not because of log inconsistency* (this can only happen if our term has passed), then you should immediately step down, and *not* update . If you do, you could race with the resetting of if you are re-elected immediately.`AppendEntries``nextIndex``nextIndex`
- A leader is not allowed to update to somewhere in a *previous* term (or, for that matter, a future term). Thus, as the rule says, you specifically need to check that . This is because Raft leaders cannot be sure an entry is actually committed (and will not ever be changed in the future) if it’s not from their current term. This is illustrated by Figure 8 in the paper.`commitIndex``log[N].term == currentTerm`

One common source of confusion is the difference between and . In particular, you may observe that , and simply not implement . This is not safe. While and are generally updated at the same time to a similar value (specifically, ), the two serve quite different purposes. is a *guess* as to what prefix the leader shares with a given follower. It is generally quite optimistic (we share everything), and is moved backwards only on negative responses. For example, when a leader has just been elected, is set to be index index at the end of the log. In a way, is used for performance – you only need to send these things to this peer.`nextIndex``matchIndex``matchIndex = nextIndex - 1``matchIndex``nextIndex``matchIndex``nextIndex = matchIndex + 1``nextIndex``nextIndex``nextIndex`

```
matchIndex` is used for safety. It is a conservative *measurement* of what prefix of the log the leader shares with a given follower. cannot ever be set to a value that is too high, as this may cause the to be moved too far forward. This is why is initialized to -1 (i.e., we agree on no prefix), and only updated when a follower *positively acknowledges* an RPC.`matchIndex``commitIndex``matchIndex``AppendEntries
```

#### [Term confusion](https://thesquareplanet.com/blog/students-guide-to-raft/#term-confusion)

Term confusion refers to servers getting confused by RPCs that come from old terms. In general, this is not a problem when receiving an RPC, since the rules in Figure 2 say exactly what you should do when you see an old term. However, Figure 2 generally doesn’t discuss what you should do when you get old RPC *replies*. From experience, we have found that by far the simplest thing to do is to first record the term in the reply (it may be higher than your current term), and then to compare the current term with the term you sent in your original RPC. If the two are different, drop the reply and return. *Only* if the two terms are the same should you continue processing the reply. There may be further optimizations you can do here with some clever protocol reasoning, but this approach seems to work well. And *not* doing it leads down a long, winding path of blood, sweat, tears and despair.

A related, but not identical problem is that of assuming that your state has not changed between when you sent the RPC, and when you received the reply. A good example of this is setting , or when you receive a response to an RPC. This is *not* safe, because both of those values could have been updated since when you sent the RPC. Instead, the correct thing to do is update to be from the arguments you sent in the RPC originally.`matchIndex = nextIndex - 1``matchIndex = len(log)``matchIndex``prevLogIndex + len(entries[])`

#### [An aside on optimizations](https://thesquareplanet.com/blog/students-guide-to-raft/#an-aside-on-optimizations)

The Raft paper includes a couple of optional features of interest. In 6.824, we require the students to implement two of them: log compaction (section 7) and accelerated log backtracking (top left hand side of page 8). The former is necessary to avoid the log growing without bound, and the latter is useful for bringing stale followers up to date quickly.

These features are not a part of “core Raft”, and so do not receive as much attention in the paper as the main consensus protocol. Log compaction is covered fairly thoroughly (in Figure 13), but leaves out some design details that you might miss if you read it too casually:

- When snapshotting application state, you need to make sure that the application state corresponds to the state following some known index in the Raft log. This means that the application either needs to communicate to Raft what index the snapshot corresponds to, or that Raft needs to delay applying additional log entries until the snapshot has been completed.

- The text does not discuss the recovery protocol for when a server crashes and comes back up now that snapshots are involved. In particular, if Raft state and snapshots are committed separately, a server could crash between persisting a snapshot and persisting the updated Raft state. This is a problem, because step 7 in Figure 13 dictates that the Raft log covered by the snapshot *must be discarded*.

  If, when the server comes back up, it reads the updated snapshot, but the outdated log, it may end up applying some log entries *that are already contained within the snapshot*. This happens since the and are not persisted, and so Raft doesn’t know that those log entries have already been applied. The fix for this is to introduce a piece of persistent state to Raft that records what “real” index the first entry in Raft’s persisted log corresponds to. This can then be compared to the loaded snapshot’s to determine what elements at the head of the log to discard.`commitIndex``lastApplied``lastIncludedIndex`

The accelerated log backtracking optimization is very underspecified, probably because the authors do not see it as being necessary for most deployments. It is not clear from the text exactly how the conflicting index and term sent back from the client should be used by the leader to determine what to use. We believe the protocol the authors *probably* want you to follow is:`nextIndex`

- If a follower does not have in its log, it should return with and .`prevLogIndex``conflictIndex = len(log)``conflictTerm = None`
- If a follower does have in its log, but the term does not match, it should return , and then search its log for the first index whose entry has term equal to .`prevLogIndex``conflictTerm = log[prevLogIndex].Term``conflictTerm`
- Upon receiving a conflict response, the leader should first search its log for . If it finds an entry in its log with that term, it should set to be the one beyond the index of the *last* entry in that term in its log.`conflictTerm``nextIndex`
- If it does not find an entry with that term, it should set .`nextIndex = conflictIndex`

A half-way solution is to just use (and ignore ), which simplifies the implementation, but then the leader will sometimes end up sending more log entries to the follower than is strictly necessary to bring them up to date.`conflictIndex``conflictTerm`

### [Applications on top of Raft](https://thesquareplanet.com/blog/students-guide-to-raft/#applications-on-top-of-raft)

When building a service on top of Raft (such as the key/value store in the [second 6.824 Raft lab](https://pdos.csail.mit.edu/6.824/labs/lab-kvraft.html), the interaction between the service and the Raft log can be tricky to get right. This section details some aspects of the development process that you may find useful when building your application.

#### [Applying client operations](https://thesquareplanet.com/blog/students-guide-to-raft/#applying-client-operations)

You may be confused about how you would even implement an application in terms of a replicated log. You might start off by having your service, whenever it receives a client request, send that request to the leader, wait for Raft to apply something, do the operation the client asked for, and then return to the client. While this would be fine in a single-client system, it does not work for concurrent clients.

Instead, the service should be constructed as a *state machine* where client operations transition the machine from one state to another. You should have a loop somewhere that takes one client operation at the time (in the same order on all servers – this is where Raft comes in), and applies each one to the state machine in order. This loop should be the *only* part of your code that touches the application state (the key/value mapping in 6.824). This means that your client-facing RPC methods should simply submit the client’s operation to Raft, and then *wait* for that operation to be applied by this “applier loop”. Only when the client’s command comes up should it be executed, and any return values read out. Note that *this includes read requests*!

This brings up another question: how do you know when a client operation has completed? In the case of no failures, this is simple – you just wait for the thing you put into the log to come back out (i.e., be passed to ). When that happens, you return the result to the client. However, what happens if there are failures? For example, you may have been the leader when the client initially contacted you, but someone else has since been elected, and the client request you put in the log has been discarded. Clearly you need to have the client try again, but how do you know when to tell them about the error?`apply()`

One simple way to solve this problem is to record where in the Raft log the client’s operation appears when you insert it. Once the operation at that index is sent to , you can tell whether or not the client’s operation succeeded based on whether the operation that came up for that index is in fact the one you put there. If it isn’t, a failure has happened and an error can be returned to the client.`apply()`

#### [Duplicate detection](https://thesquareplanet.com/blog/students-guide-to-raft/#duplicate-detection)

As soon as you have clients retry operations in the face of errors, you need some kind of duplicate detection scheme – if a client sends an to your server, doesn’t hear back, and re-sends it to the next server, your function needs to ensure that the isn’t executed twice. To do so, you need some kind of unique identifier for each client request, so that you can recognize if you have seen, and more importantly, applied, a particular operation in the past. Furthermore, this state needs to be a part of your state machine so that all your Raft servers eliminate the *same* duplicates.`APPEND``apply()``APPEND`

There are many ways of assigning such identifiers. One simple and fairly efficient one is to give each client a unique identifier, and then have them tag each request with a monotonically increasing sequence number. If a client re-sends a request, it re-uses the same sequence number. Your server keeps track of the latest sequence number it has seen for each client, and simply ignores any operation that it has already seen.

#### [Hairy corner-cases](https://thesquareplanet.com/blog/students-guide-to-raft/#hairy-corner-cases)

If your implementation follows the general outline given above, there are at least two subtle issues you are likely to run into that may be hard to identify without some serious debugging. To save you some time, here they are:

**Re-appearing indices**: Say that your Raft library has some method that takes a command, and return the index at which that command was placed in the log (so that you know when to return to the client, as discussed above). You might assume that you will never see return the same index twice, or at the very least, that if you see the same index again, the command that first returned that index must have failed. It turns out that neither of these things are true, even if no servers crash.`Start()``Start()`

Consider the following scenario with five servers, S1 through S5. Initially, S1 is the leader, and its log is empty.

1. Two client operations (C1 and C2) arrive on S1
2. `Start()` return 1 for C1, and 2 for C2.
3. S1 sends out an to S2 containing C1 and C2, but all its other messages are lost.`AppendEntries`
4. S3 steps forward as a candidate.
5. S1 and S2 won’t vote for S3, but S3, S4, and S5 all will, so S3 becomes the leader.
6. Another client request, C3 comes in to S3.
7. S3 calls (which returns 1)`Start()`
8. S3 sends an to S1, who discards C1 and C2 from its log, and adds C3.`AppendEntries`
9. S3 fails before sending to any other servers.`AppendEntries`
10. S1 steps forward, and because its log is up-to-date, it is elected leader.
11. Another client request, C4, arrives at S1
12. S1 calls , which returns 2 (which was also returned for .`Start()``Start(C2)`
13. All of S1’s are dropped, and S2 steps forward.`AppendEntries`
14. S1 and S3 won’t vote for S2, but S2, S4, and S5 all will, so S2 becomes leader.
15. A client request C5 comes in to S2
16. S2 calls , which returns 3.`Start()`
17. S2 successfully sends to all the servers, which S2 reports back to the servers by including an updated in the next heartbeat.`AppendEntries``leaderCommit = 3`

Since S2’s log is , this means that the entry that committed (and was applied at all servers, including S1) at index 2 is C2. This despite the fact that C4 was the last client operation to have returned index 2 at S1.`[C1 C2 C5]`

**The four-way deadlock**: All credit for finding this goes to [Steven Allen](http://stebalien.com/), another 6.824 TA. He found the following nasty four-way deadlock that you can easily get into when building applications on top of Raft.

Your Raft code, however it is structured, likely has a -like function that allows the application to add new commands to the Raft log. It also likely has a loop that, when is updated, calls on the application for every element in the log between and . These routines probably both take some lock . In your Raft-based application, you probably call Raft’s function somewhere in your RPC handlers, and you have some code somewhere else that is informed whenever Raft applies a new log entry. Since these two need to communicate (i.e., the RPC method needs to know when the operation it put into the log completes), they both probably take some lock .`Start()``commitIndex``apply()``lastApplied``commitIndex``a``Start()``b`

In Go, these four code segments probably look something like this:

```
func (a *App) RPC(args interface{}, reply interface{}) {
    // ...
    a.mutex.Lock()
    i := a.raft.Start(args)
    // update some data structure so that apply knows to poke us later
    a.mutex.Unlock()
    // wait for apply to poke us
    return
}
func (r *Raft) Start(cmd interface{}) int {
    r.mutex.Lock()
    // do things to start agreement on this new command
    // store index in the log where cmd was placed
    r.mutex.Unlock()
    return index
}
func (a *App) apply(index int, cmd interface{}) {
    a.mutex.Lock()
    switch cmd := cmd.(type) {
    case GetArgs:
        // do the get
	// see who was listening for this index
	// poke them all with the result of the operation
    // ...
    }
    a.mutex.Unlock()
}
func (r *Raft) AppendEntries(...) {
    // ...
    r.mutex.Lock()
    // ...
    for r.lastApplied < r.commitIndex {
      r.lastApplied++
      r.app.apply(r.lastApplied, r.log[r.lastApplied])
    }
    // ...
    r.mutex.Unlock()
}
```

Consider now if the system is in the following state:

- `App.RPC` has just taken and called `a.mutex``Raft.Start`
- `Raft.Start` is waiting for `r.mutex`
- `Raft.AppendEntries` is holding , and has just called `r.mutex``App.apply`

We now have a deadlock, because:

- `Raft.AppendEntries` won’t release the lock until returns.`App.apply`
- `App.apply` can’t return until it gets .`a.mutex`
- `a.mutex` won’t be released until returns.`App.RPC`
- `App.RPC` won’t return until returns.`Raft.Start`
- `Raft.Start` can’t return until it gets .`r.mutex`
- `Raft.Start` has to wait for .`Raft.AppendEntries`

There are a couple of ways you can get around this problem. The easiest one is to take *after* calling in . However, this means that may be called for the operation that just called on *before* has a chance to record the fact that it wishes to be notified. Another scheme that may yield a neater design is to have a single, dedicated thread calling from . This thread could be notified every time is updated, and would then not need to hold a lock in order to apply, breaking the deadlock.`a.mutex``a.raft.Start``App.RPC``App.apply``App.RPC``Raft.Start``App.RPC``r.app.apply``Raft``commitIndex`

### 中文翻译

在过去的几个月里，我一直是麻省理工学院[6.824分布式系统](https://pdos.csail.mit.edu/6.824/)课程的助教。 传统上，该班级有许多实验室建立在Paxos上 共识算法，但今年，我们决定转向[Raft](https://raft.github.io/)。Raft 被“设计得易于 理解“，我们希望这种变化可能会使学生 生活更轻松。

这篇文章，以及随附的[《木筏](https://thesquareplanet.com/blog/instructors-guide-to-raft/)指南》帖子，记录了我们的旅程 与 Raft 一起使用，并希望对 Raft 的实现者有用 协议和学生试图更好地了解Raft的 内部。如果您正在寻找Paxos与Raft的比较，或者 更多关于木筏的教学分析，你应该去阅读讲师的 指导。这篇文章的底部包含通常的问题列表 由6.824名学生提出，以及这些问题的答案。如果你 遇到本文主要内容中未列出的问题， 查看[问答](https://thesquareplanet.com/blog/raft-qa/)。该帖子是 相当长，但它提出的所有观点都是很多真正的问题 6.824名学生（和助教）遇到了。这是一本值得一读的书。

### [背景](https://thesquareplanet.com/blog/students-guide-to-raft/#background)

在我们深入研究 Raft 之前，一些上下文可能会有用。6.824 曾经有 一组[基于 Paxos 的 ](http://nil.csail.mit.edu/6.824/2015/labs/lab-3.html)实验室 内置[围棋](https://golang.org/);选择 Go 既因为它是 易于学生学习，并且因为非常适合 编写并发的分布式应用程序（goroutines 进来了 特别方便）。在四个实验过程中，学生构建一个 容错分片键值存储。第一个实验室让他们建造了一个 基于共识的日志库，第二个在 那个，第三个在多个容错之间分片了密钥空间 集群，具有容错分片主处理配置 变化。我们还有第四个实验室，学生必须在其中处理 有磁盘和无磁盘的计算机的故障和恢复 完整。此实验室可作为学生的默认最终项目使用。

今年，我们决定使用 Raft 重写所有这些实验室。第一个 三个实验室都是一样的，但第四个实验室被放弃了 持久性和故障恢复已经内置到 Raft 中。这 文章将主要讨论我们在第一个实验室的经验，因为它是 与 Raft 最直接相关的一个，尽管我也会谈到 在 Raft 之上构建应用程序（如第二个实验室）。

筏子，对于那些刚刚了解它的人来说，是最好的 由协议网络上的文本描述[ 网站](https://raft.github.io/)：

> Raft 是一种共识算法，旨在轻松 理解。它相当于Paxos的容错和 性能。不同的是，它被分解成相对 独立的子问题，它干净地解决了所有主要部分 实际系统需要。我们希望Raft能够达成共识 可供更广泛的受众使用，并且这些更广泛的受众将是 能够开发各种更高质量的基于共识的系统 比今天可用的。

像[这样的](http://thesecretlivesofdata.com/raft/)可视化很好地概述了协议的主要组成部分，并且 这篇论文很好地说明了为什么需要各种部分。如果 你还没有读过[扩展的木筏 纸](https://raft.github.io/raft.pdf)，你应该去读 在继续本文之前，因为我将假设一个相当熟悉 与木筏。

与所有分布式共识协议一样，魔鬼非常在 细节。在没有故障的稳定状态下，Raft 的 行为易于理解，并且可以直观地解释 方式。例如，从可视化中很容易看出， 假设没有失败，最终将选出一位领导者，并且 最终，发送给领导者的所有操作都将由 关注者顺序正确。但是，当消息延迟时，网络 引入了分区和故障服务器，每个如果，但是， 而且，变得至关重要。特别是，有许多错误 我们看到一遍又一遍地重复，仅仅是由于误解或 阅读论文时的疏忽。这个问题并非Raft所独有， 并且是所有复杂的分布式系统中出现的一个，这些系统提供 正确性。

### [实现筏](https://thesquareplanet.com/blog/students-guide-to-raft/#implementing-raft)

Raft 的最终指南在 Raft 论文的图 2 中。这个数字 指定在 Raft 服务器之间交换的每个 RPC 的行为， 给出服务器必须维护的各种不变量，并指定何时 应执行某些操作。我们将*在本文*的其余部分大量讨论图 2。它需要一*字不差*地遵循。

图 2 定义了每个服务器在各种状态下应该对每个 传入的 RPC，以及何时发生某些其他事情（例如 就像在日志中应用条目是安全的一样）。起初，您可能是 试图将图 2 视为一种非正式指南;你读过它 一次，然后开始编写一个大致遵循的实现 它说要做什么。这样做，您将快速启动并运行 一个主要有效的 Raft 实现。然后问题开始了。

事实上，图 2 非常精确，每一条语句 在规范术语中，它应该被视为**必须**，而不是**应该**。例如，您可以合理地重置对等方的选举 计时器，只要您收到 或 RPC，作为 两者都表明其他同行要么认为自己是领导者，要么是 努力成为领导者。直觉上，这意味着我们不应该 是干扰。但是，如果您仔细阅读图 2，它会说：`AppendEntries``RequestVote`

> 如果选举超时过去而没有收到*当前领导人的* RPC 或授予候选人*投票*权：转换为 候选人。`AppendEntries`

事实证明，区别很重要，因为前一种实现 在某些情况下，可能导致活性显着降低。

#### [细节的重要性](https://thesquareplanet.com/blog/students-guide-to-raft/#the-importance-of-details)

为了使讨论更具体，让我们考虑一个例子 绊倒了6.824名学生。Raft论文在许多地方提到了*心跳RPC*。具体来说，领导者将 偶尔（每个检测信号间隔至少一次）向所有对等方发送 RPC，以防止它们启动新的 选举。如果领导者没有要发送到特定对等方的新条目， RPC 不包含任何条目，被视为 心跳。`AppendEntries``AppendEntries`

我们的许多学生认为心跳在某种程度上是“特别的”; 当对等方收到心跳时，它应该以不同的方式对待它 来自非检测信号 RPC。特别是，许多人会 只需在收到心跳时重置他们的选举计时器，然后 然后返回成功，而不执行 中指定的任何检查 图2.这是*极其危险的*。通过接受 RPC， 追随者隐式地告诉领导者他们的日志与 领导者的登录并包括参数中包含的内容。收到回复后，领导可能会 然后（错误地）确定某个条目已被复制到 大多数服务器，并开始提交它。`AppendEntries``prevLogIndex``AppendEntries`

许多人遇到的另一个问题（通常在解决上述问题后立即）， 是，在收到心跳时，他们会截断追随者的 记录以下 ，然后附加 中包含的任何条目 论点。*这也是*不正确的。我们可以一次 再次转到图 2：`prevLogIndex``AppendEntries`

> *如果*现有条目与新条目冲突（相同的索引但 不同的条款），删除现有条目及其后面的所有条目。

*这里的如果*至关重要。如果追随者拥有所有条目，则领导者 发送后，追随者**不得**截断其日志。==**必须**保留领导者发送的条目*之后*的任何元素==。这是 因为我们可能从 领导者，截断日志将意味着“收回”我们 可能已经告诉领导我们的日志。`AppendEntries`

### [调试筏](https://thesquareplanet.com/blog/students-guide-to-raft/#debugging-raft)

不可避免地，Raft 实现的第一次迭代将是 马车。第二个也是如此。第三。第四。一般来说，每一个 会比前一个少，而且根据经验，大部分 您的错误将是未忠实遵循图 2 的结果。

在调试时，Raft，通常有四个主要的错误来源： 活锁、不正确或不完整的 RPC 处理程序、未能遵循 规则和术语混淆。死锁也是一个常见问题，但它们 通常可以通过记录所有锁和解锁来调试，并且 弄清楚你正在采取哪些锁，但不释放。让我们 依次考虑以下每个：

#### [活锁](https://thesquareplanet.com/blog/students-guide-to-raft/#livelocks)

当系统活锁时，系统中的每个节点都在执行 一些东西，但总的来说，你的节点处于这样的状态，没有 正在取得进展。这在 Raft 中很容易发生， 特别是如果您不虔诚地遵循图 2。一个活锁 场景特别频繁出现;没有领导人被选举出来，或者一次 一个领导者被选举出来，另一个节点开始选举，迫使 最近当选的领导人立即退位。

出现这种情况的原因有很多，但有一个 我们看到许多学生犯的一些错误：

- 确保在图 2 显示时*准确*重置选举计时器 你应该。具体来说，您应该*只*重新开始选举 计时器，如果 a） 您从*当前*领导者那里获得 RPC （即，如果参数中的术语已过时，则 *不应*重置计时器）;b） 您正在开始选举;或 c） 您*向*其他同行授予投票权。`AppendEntries``AppendEntries`

  最后一种情况在不可靠的网络中尤其重要，其中 关注者可能有不同的日志;在这些情况下， 您通常最终只会得到少量的服务器 大多数服务器都愿意投票支持。如果重置 每当有人要求您投票给他们时，选举计时器都会使 日志过时的服务器同样有可能向前迈进 至于日志较长的服务器。

  事实上，因为服务器太少了，有足够的 最新的日志，这些服务器不太可能能够保存 在足够和平的情况下进行选举。如果您遵循规则 从图 2 中，具有最新日志的服务器不会 被过时的服务器选举打断，因此更有可能 完成选举并成为领导者。

- 按照图 2 的说明操作，了解何时应开始选举。 特别要注意的是，如果您是候选人（即，您 当前正在进行选举），但选举计时器触发，您 应该开始*另一次*选举。这对于避免 由于 RPC 延迟或丢弃而导致系统停止。

- 在处理传入的 RPC *之前*，请确保遵循“服务器规则”中的第二条规则。第二条规则规定：

  > 如果 RPC 请求或响应包含术语 ： set ，则转换为追随者 （§5.1）`T > currentTerm``currentTerm = T`

  例如，如果您已经在当前任期内投票，并且 传入的RPC有一个更高的术语，你应该*首先*下台并采用他们的术语（从而重置），*然后*处理RPC，这将导致你 授予投票！`RequestVote``votedFor`

#### [不正确的 RPC 处理程序](https://thesquareplanet.com/blog/students-guide-to-raft/#incorrect-rpc-handlers)

尽管图 2 准确地说明了每个 RPC 处理程序应该执行的操作， 一些微妙之处仍然很容易被忽略。以下是我们保留的一些 一遍又一遍地看到，你应该留意 您的实现：

- 如果步骤显示“回复错误”，这意味着您应该*回复 立即*，并且不执行任何后续步骤。
- 如果你得到一个带有该分数的 RPC 在日志末尾之后，您应该像处理日志一样处理它 确实有该条目，但该术语不匹配（即回复错误）。`AppendEntries``prevLogIndex`
- RPC 处理程序的检查 2 *应执行 如果领导者没有发送任何条目*。`AppendEntries`
- 在最后一步是*必要的*， 并且需要使用最后一个*新*条目的索引进行计算。 仅仅拥有适用的*功能是不够的* 日志中的内容介于和停止之间 当它到达日志末尾时。这是因为您可能有 日志中与*领导者日志不同的*条目在 领导者发送给您的条目（这些条目都与您的条目匹配 日志）。因为 #5 规定您只有在*以下情况下*才会截断日志 有冲突的条目，这些条目不会被删除，如果超出领导发送给您的条目，您可以 应用不正确的条目。`min``AppendEntries``lastApplied``commitIndex``leaderCommit`
- 实施“最新日志*”检查非常重要* 在第 5.4 节中描述。没有作弊，只是检查长度！

#### [不遵守规则](https://thesquareplanet.com/blog/students-guide-to-raft/#failure-to-follow-the-rules)

虽然 Raft 论文非常明确地说明了如何实现每个 RPC 处理程序，它还留下了许多规则的实现和 未指定的不变量。这些列在“服务器规则”中 图 2 右侧的块。虽然其中一些是公平的 不言自明，也有一些需要设计你的 非常小心地申请，以免违反规则：

- 如果在执行过程中*的任何时候*，您 应应用特定的日志条目。你这样做并不重要 直接（例如，在 RPC 处理程序中），但是 *请务必确保*仅完成此应用程序 由一个实体。具体来说，您需要有一个专门的 “应用器”，或者锁定这些应用，以便其他一些 例程不会同时检测到需要应用条目，并且 尝试申请。`commitIndex > lastApplied``AppendEntries`
- 确保检查以下任一 定期更新，或更新后（即更新后）。例如，如果您检查 在发送给同行的同时，您可能有 等到*下一个*条目追加到日志中后再应用 您刚刚发送并得到确认的条目。`commitIndex > lastApplied``commitIndex``matchIndex``commitIndex``AppendEntries`
- 如果领导者发出 RPC，并且被拒绝，但不*是因为日志不一致*（这只有在我们的术语中才会发生 已通过），那么您应该立即下台，*而不是*更新.如果你这样做，你可以立即重置你是否连任。`AppendEntries``nextIndex``nextIndex`
- 领导者不允许更新到*上一*任期（或就此而言，未来任期）的某个地方。因此，作为 规则说，你特别需要检查.这是因为 Raft 领导者无法确定条目是否 实际提交（并且将来永远不会更改）如果 这不是他们目前的任期。图 8 对此进行了说明 论文。`commitIndex``log[N].term == currentTerm`

混淆的一个常见来源是 和 之间的区别。特别是，您可能会观察到 ，并且干脆不实现 。这是不安全的。 虽然 和 通常同时更新 时间到相似值（具体来说， ）， 两者的目的完全不同。 是一个*猜测* 领导者与给定追随者共享的前缀。一般是 相当乐观（我们分享一切），并且仅在 负面回应。例如，当领导者刚刚当选时，被设置为日志末尾的索引索引。在某种程度上，用于性能 - 您只需要发送这些 对这个同行的事情。`nextIndex``matchIndex``matchIndex = nextIndex - 1``matchIndex``nextIndex``matchIndex``nextIndex = matchIndex + 1``nextIndex``nextIndex``nextIndex`

```
matchIndex`用于安全。这是*对* 领导者与给定关注者共享的日志前缀。 永远不能设置为太高的值，因为这可能会 导致 向前移动太远。这就是为什么初始化为 -1（即，我们同意没有前缀），并且 仅当关注者肯定地*确认* RPC 时才会更新。`matchIndex``commitIndex``matchIndex``AppendEntries
```

#### [术语混淆](https://thesquareplanet.com/blog/students-guide-to-raft/#term-confusion)

术语混淆是指服务器被来自的 RPC 混淆 旧术语。一般来说，这在接收 RPC 时不是问题， 因为图 2 中的规则准确地说明了当您看到时应该执行的操作 一个古老的术语。但是，图 2 通常不讨论您应该做什么 当您收到旧的 RPC *回复*时执行。根据经验，我们发现 到目前为止，最简单的方法是首先在回复中记录该术语 （它可能高于您当前的期限），然后比较 当前术语，其中包含您在原始 RPC 中发送的术语。如果两者是 不同，删除回复并返回。*仅*当这两个项是 如果您继续处理回复，也是如此。可能还有进一步 您可以在此处使用一些聪明的协议推理进行优化，但是 这种方法似乎效果很好。*不*这样做会导致漫长的， 血汗、泪水和绝望的曲折之路。

一个相关但不完全相同的问题是假设你的状态 在发送 RPC 和收到 答。一个很好的例子是设置 ， 或者当您收到对 RPC 的响应时。这 *不安全*，因为这两个值都可以更新 从您发送 RPC 开始。相反，正确的做法是从参数更新 您最初发送了 RPC。`matchIndex = nextIndex - 1``matchIndex = len(log)``matchIndex``prevLogIndex + len(entries[])`

#### [关于优化的旁白](https://thesquareplanet.com/blog/students-guide-to-raft/#an-aside-on-optimizations)

Raft 论文包括一些感兴趣的可选功能。在 6.824，我们要求学生实现其中两个：日志压缩 （第 7 节）和加速日志回溯（页面左上角） 8）.前者是避免原木无限制生长所必需的，并且 后者对于快速更新过时的关注者很有用。

这些功能不是“核心 Raft”的一部分，因此不会收到 作为主要的共识协议，论文中备受关注。日志 压缩被相当彻底地覆盖（在图 13 中），但遗漏了 如果你太随意地阅读它，你可能会错过一些设计细节：

- 快照应用程序状态时，需要确保 应用程序状态对应于某个已知索引之后的状态 在木筏日志中。这意味着应用程序需要 向 Raft 传达快照对应的索引，或者 Raft 需要延迟应用其他日志条目，直到 快照已完成。

- 本文未讨论服务器何时的恢复协议 崩溃，现在涉及快照后恢复。在 特别是，如果 Raft 状态和快照是分开提交的， 服务器可能会在持久保存快照和持久保存 更新了筏状态。这是一个问题，因为图 7 中的步骤 13 规定快照覆盖的 Raft 日志*必须 丢弃*。

  如果当服务器恢复时，它会读取更新的快照，但是 过时的日志，它最终可能会应用一些*日志条目 已包含在快照中*。发生这种情况是因为 和 没有持久化，所以 Raft 不知道这些日志条目是否已应用。这 ==修复方法是向 Raft 引入一段持久状态 记录 Raft 持久日志中第一个条目的“真实”索引 对应==。然后可以将其与加载的快照进行比较，以确定日志头部的哪些元素 丢弃。`commitIndex``lastApplied``lastIncludedIndex`

加速日志回溯优化非常不明确， 可能是因为作者认为这对大多数人来说不是必需的 部署。从文本中不清楚冲突的确切方式 领导者应使用从客户端发回的索引和术语来 确定要使用的内容。我们相信作者*可能*希望您遵循的协议是：`nextIndex`

- 如果关注者的日志中没有，它应该 返回 和 。`prevLogIndex``conflictIndex = len(log)``conflictTerm = None`
- 如果关注者的日志中确实有，但该术语有 不匹配，它应该返回 ， 然后在其日志中搜索条目具有术语的第一个索引 等于 。`prevLogIndex``conflictTerm = log[prevLogIndex].Term``conflictTerm`
- 收到冲突回复后，领导者应首先搜索 它的日志为 。如果它在其日志中找到一个条目，则包含该条目 术语，它应设置为超出其日志中该术语中*最后一个*条目的索引的索引。`conflictTerm``nextIndex`
- 如果找不到带有该术语的条目，则应设置 .`nextIndex = conflictIndex`

一个折衷的解决方案是只使用（和忽略），这简化了实现，但随后 领导者有时会向追随者发送更多日志条目 而不是使它们保持最新状态的绝对必要条件。`conflictIndex``conflictTerm`

### [筏子顶部的应用](https://thesquareplanet.com/blog/students-guide-to-raft/#applications-on-top-of-raft)

在 Raft 上构建服务时（例如 [第二个 6.824 筏 实验室](https://pdos.csail.mit.edu/6.824/labs/lab-kvraft.html)， 服务和 Raft 日志之间的交互可能很难获得 右。本节详细介绍了开发过程的某些方面： 在构建应用程序时，您可能会发现很有用。

#### [应用客户端操作](https://thesquareplanet.com/blog/students-guide-to-raft/#applying-client-operations)

您可能会对如何在 复制日志的条款。您可以从获得服务开始， 每当它收到客户端请求时，将该请求发送给领导者， 等待 Raft 应用一些东西，做客户要求的操作， ，然后返回到客户端。虽然这在 单客户端系统，它不适用于并发客户端。

相反，服务应该构造为*状态机*，其中 客户端操作将计算机从一种状态转换为另一种状态。你 应该在某个地方有一个循环，当时需要一个客户端操作 （在所有服务器上以相同的顺序 - 这就是 Raft 的用武之地），以及 按顺序将每个应用于状态机。此循环应该是代码中*唯一*接触应用程序状态的部分（ 6.824 中的键/值映射）。这意味着面向客户端的 RPC 方法应该简单地将客户端的操作提交给 Raft，然后*等待*这个“applier 循环”应用该操作。只 ==当客户端的命令出现时，它应该被执行，并且任何返回 读出值。请注意，*这包括读取请求*！==

这就引出了另一个问题：==你怎么知道客户端何时操作 已完成==？在没有失败的情况下，这很简单——你只是 等待您放入日志的内容重新出来（即 传递给 ）。发生这种情况时，您将结果返回到 客户。但是，如果出现故障会发生什么？例如，您 当客户最初与您联系时，可能是领导者，但是 其他人已经当选，而您提出的客户请求 日志已被丢弃。显然，您需要让客户尝试 同样，但是您如何知道何时告诉他们错误？`apply()`

解决此问题的一种简单方法是==记录 Raft 日志中的位置 插入客户端时将显示客户端的操作==。一旦操作在 该索引被发送到 ，您可以判断是否 客户端的操作是否成功取决于出现的操作是否成功 因为该索引实际上是您放在那里的索引。如果不是，则失败 已发生，可以向客户端返回错误。`apply()`

#### [重复检测](https://thesquareplanet.com/blog/students-guide-to-raft/#duplicate-detection)

一旦客户端在出现错误时重试操作，您就会 需要某种重复检测方案 - 如果客户端向您的服务器发送一个，没有收到回复，并将其重新发送到下一个 服务器，您的函数需要确保不是 执行两次。为此，您需要某种==唯一标识符 每个客户端请求==，以便您可以识别您是否看到过，以及 更重要的是，应用了过去的特定操作。 此外，此状态需要成为状态机的一部分，以便 所有Raft服务器都==消除了*相同的*重复项==。`APPEND``apply()``APPEND`

有许多方法可以分配此类标识符。一个简单而公平 有效的方法是给==每个客户端一个唯一的标识符==，然后有 它们用==单调递增的序列号标记每个请求==。 如果客户端重新发送请求，它将重新使用相同的序列号。 您的服务器会跟踪它看到的最新序列号 每个客户端，并简单地忽略它已经看到的任何操作。

#### [毛茸茸的角落案例](https://thesquareplanet.com/blog/students-guide-to-raft/#hairy-corner-cases)

如果您的实现遵循上面给出的一般概述，则有 至少有两个微妙的问题，你可能会遇到 如果没有一些认真的调试，很难识别。为了节省一些时间， 他们在这里：

**重新出现的索引**： 假设您的 Raft 库有一些方法需要 命令，并返回该命令放置在 日志（以便您知道何时返回到客户端，如上所述）。 您可能会假设您永远不会看到返回相同的索引 两次，或者至少，如果您再次看到相同的索引， 首先返回该索引的命令必须失败。事实证明 即使没有服务器崩溃，这些都不是真的。`Start()``Start()`

请考虑以下具有五台服务器（S1 到 S5）的方案。 最初，S1 是领导者，其日志为空。

1. 两个客户端操作（C1 和 C2）到达 S1
2. `Start()`对于 C1，返回 1，为 C2 返回 2。
3. S1 向 S2 发送包含 C1 和 C2 的 A，但所有 它的其他消息将丢失。`AppendEntries`
4. S3 作为候选项向前迈进。
5. S1 和 S2 不会投票给 S3，但 S3、S4 和 S5 都会投票，所以 S3 成为领导者。
6. 另一个客户端请求 C3 进入 S3。
7. S3 调用（返回 1）`Start()`
8. S3 向 S1 发送 a，S1 从其 日志，并添加 C2。`AppendEntries`
9. S3 在发送到任何其他服务器之前失败。`AppendEntries`
10. S1 向前迈进，由于其日志是最新的，因此它被选中 领导。
11. 另一个客户端请求 C4 到达 S1
12. S1 调用 ，返回 2（也返回了 。`Start()``Start(C2)`
13. 所有 S1 都被丢弃，S2 向前迈进。`AppendEntries`
14. S1 和 S3 不会投票给 S2，但 S2、S4 和 S5 都会投票，所以 S2 成为领导者。
15. 客户端请求 C5 进入 S2
16. S2 调用 ，返回 3。`Start()`
17. S2 成功发送到所有服务器，S2 通过在下一个检测信号中包含更新来向服务器报告。`AppendEntries``leaderCommit = 3`

由于 S2 的日志是 ，这意味着提交 （并应用于所有服务器，包括 S1）在索引 2 处是 C2。这 尽管 C4 是最后一个返回的客户端操作 索引 2 位于 S1。`[C1 C2 C5]`

**四向僵局**： 找到这个的所有功劳都归功于[史蒂文 艾伦](http://stebalien.com/)，另一个 6.824 TA。他发现以下内容 令人讨厌的四向死锁，您在构建时很容易陷入 Raft 之上的应用程序。

你的 Raft 代码，无论它是如何结构化的，都可能有一个类似 允许应用程序向 Raft 添加新命令的功能 .log。它还可能有一个循环，当更新时， 针对 和 之间的日志中的每个元素调用应用程序。这些例程可能都需要一些 锁。在基于 Raft 的应用程序中，您可能会在 RPC 处理程序中的某个位置调用 Raft 的函数，并且您有一些 在其他地方的代码，每当 Raft 应用新日志时都会收到通知 进入。由于这两者需要通信（即，RPC 方法需要 知道它放入日志的操作何时完成），它们都 可能会采取一些锁定.`Start()``commitIndex``apply()``lastApplied``commitIndex``a``Start()``b`

在 Go 中，这四个代码段可能如下所示：

```go
func (a *App) RPC(args interface{}, reply interface{}) {
    // ...
    a.mutex.Lock()
    i := a.raft.Start(args)
    // update some data structure so that apply knows to poke us later
    a.mutex.Unlock()
    // wait for apply to poke us
    return
}
func (r *Raft) Start(cmd interface{}) int {
    r.mutex.Lock()
    // do things to start agreement on this new command
    // store index in the log where cmd was placed
    r.mutex.Unlock()
    return index
}
func (a *App) apply(index int, cmd interface{}) {
    a.mutex.Lock()
    switch cmd := cmd.(type) {
    case GetArgs:
        // do the get
	// see who was listening for this index
	// poke them all with the result of the operation
    // ...
    }
    a.mutex.Unlock()
}
func (r *Raft) AppendEntries(...) {
    // ...
    r.mutex.Lock()
    // ...
    for r.lastApplied < r.commitIndex {
      r.lastApplied++
      r.app.apply(r.lastApplied, r.log[r.lastApplied])
    }
    // ...
    r.mutex.Unlock()
}
```

现在考虑系统是否处于以下状态：

- `App.RPC`刚刚接过电话`a.mutex``Raft.Start`
- `Raft.Start`正在等待`r.mutex`
- `Raft.AppendEntries`正在憧憬，刚刚叫了`r.mutex``App.apply`

我们现在陷入僵局，因为：

- `Raft.AppendEntries`在返回之前不会释放锁。`App.apply`
- `App.apply`在得到之前不能返回.`a.mutex`
- `a.mutex`在返回之前不会发布。`App.RPC`
- `App.RPC`在返回之前不会返回。`Raft.Start`
- `Raft.Start`在得到之前不能返回.`r.mutex`
- `Raft.Start`必须等待.`Raft.AppendEntries`

有几种方法可以解决此问题。最简单的 一种是打电话*后*接受. 但是，这意味着可能会调用该操作 *之前*刚刚调用的有一个 有机会记录它希望收到通知的事实。 另一种可能产生更整洁设计的方案是有一个单一的， 从 调用专用线程。此线程可能是 每次更新都会通知，然后不需要 按住锁才能应用，打破僵局。`a.mutex``a.raft.Start``App.RPC``App.apply``App.RPC``Raft.Start``App.RPC``r.app.apply``Raft``commitIndex`