# 6.824-RaftLab-Part2D-Development Quicknotes


### Overview

> This topic can be found under [Lab2 instruction](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html).

Modifying Raft to cooperate with services that persistently store a "snapshot" of their state from time to time, at which point Raft discards log entries that precede the snapshot.

It's now possible for a follower to fall so far behind that the leader has discarded the log entries it needs to catch up; the **leader must then send a snapshot plus the log starting at the time** of the snapshot.

diagram of Raft interactions

{{< image src="https://image.p1nant0m.com/raft.png" caption="KeyValue-Store Server Architecture" src_s="https://image.p1nant0m.com/raft.png" src_l="https://image.p1nant0m.com/raft.png" >}}

Raft must provide the following function that the service can call with a serialized snapshot of its state. In this way, Raft can discard the log entries safely preceding this `Snapshot()`

`Snapshot(index int, snapshot []byte)`

The index argument indicates the highest log entry that's reflected in the snapshot. Raft should discard its log entries before that point. You'll need to revise your Raft code to operate while storing only the tail of the log.

You'll need to implement the InstallSnapshot RPC discussed in the paper that allows a Raft leader to tell a lagging Raft peer to replace its state with a snapshot. **You will likely need to think through how InstallSnapshot should interact with the state and rules in Figure 2.**

> Changing original Code to adopt the new requirements in Part2D. This happens in making condition judgments when you determine whether it is the correct time to change Raft State.
> 

When a follower's Raft code receives an InstallSnapshot RPC, it can use the applyCh
 to send the snapshot to the service in an ApplyMsg.

Your Raft should persist both Raft state and the corresponding snapshot. Use `persister. SaveStateAndSnapshot()`, which takes separate arguments for the Raft state and the corresponding snapshot. If there's no snapshot, pass `nil`  as the snapshot argument.

Previously, this lab recommended that you implement a function called CondInstallSnapshot to avoid the requirement that snapshots and log entries sent on applyCh are coordinated. This vestigial API interface remains, but you are discouraged from implementing it: instead, we suggest that you simply have it `return true`.

**Tips:**

- A good place to start is to modify your code so that it is able to store just the part of the log starting at some index X. Initially you can set X to zero and run the 2B/2C tests. Then make Snapshot(index) discard the log before index, and set X equal to index. If all goes well you should now pass the first 2D test.
- You won't be able to store the log in a Go slice and use Go slice indices interchangeably with Raft log indices; you'll need to index the slice in a way that accounts for the discarded portion of the log.
- Next: have the leader send an InstallSnapshot RPC if it doesn't have the log entries required to bring a follower up to date.
- Send the entire snapshot in a single InstallSnapshot RPC. Don't implement Figure 13's offset mechanism for splitting up the snapshot.
- Raft must discard old log entries in a way that allows the Go garbage collector to free and re-use the memory; this requires that there be no reachable references (pointers) to the discarded log entries.
- Even when the log is trimmed, your implementation still needs to properly send the term and index of the entry prior to new entries in AppendEntries RPCs; this may require saving and referencing the latest snapshot's lastIncludedTerm/lastIncludedIndex (consider whether this should be persisted).
- A reasonable amount of time to consume for the full set of Lab 2 tests (2A+2B+2C+2D) without -race is 6 minutes of real-time and one minute of CPU time. When running with -race, it is about 10 minutes of real-time and two minutes of CPU time.
- Figure13 shows the parameters that you may be used in implementing InstallSnapshotRPCs


{{< image src="https://image.p1nant0m.com/figure13.png" caption="Figure 13" src_s="https://image.p1nant0m.com/figure13.png" src_l="https://image.p1nant0m.com/figure13.png">}}

### Implementation

😭 So many concurrency problems, headaches.

- [x]  Modify `Apply()`  in the form of receiving a signal via notifyApply channel
- [x]  Implement Snapshot but Not installSnapshotRPC and that should pass the First Test
    - Modify the `apis.go`  in order to add some helpful function to maintain the state of Snapshot
    - Modify the `getRangeEntries()/getEntry()`  which should be take offset into consideration when adding snapshot functionality. `getRangeEntries()`  takes `fromidx` and `toidx` as its parameters, which should be minus `lastSnapshotIndex` to get the correct position in the log array index. (I should carefully check whether this modification will lead to disaster🥲)
    - Separate the implementation in `persist()`  and create a new function to retrieve serialization data of raftState
    - Add `lastSnapshotIndex` and `lastSnapshotTerm` to `persist()` and `readPersist()`
    - Modify the `logReplicate()` . When sending appendentriesRPC call, we should check the `nextIndex[peerIdx]` whether is smaller than `lastSnapshotIdx`. If yes, we should send IntallSnapshotRPC rather than sending appendEntriesRPC. Moreover, we update the nextIndex[peerIdx] based on the peer’s reply which contains the conflict Index, and it will reach the condition that we need to send InstallSnapshotRPC. There are two approaches to handle this situation: 1) Send InstallSnapshotRPC whenever it reaches the condition in peer’s reply  OR 2) **Send InstallSnapshotRPC in the next `logReplicate()` procedure.**
        - logReplicate is called when the leader’s heartBeat timer is a timeout and sending RPCs are in parallel, thus the `logReplicate()` may be called during the previous one still keep running. That may be problematic. ~~Whatever, I add a synchronized mechanism to avoid potential bugs here. 😫~~
- [x]  InstallSnapshot RPC
    - Implement `installSnapshot()` and define the data structure of request/response using in this call.
    - Implement `InstallSnapshotRPC` `
        - Symptoms: failed to reach an agreement, one server can not catch up with others. And it tries to become a leader but the remote server r has a more up-to-date log, so it fails to be elected.
        - Diagnosis: 1) Reading What `TestSnapshotInstall2D` do
            - The test tries to reconnect the server which has been disconnected in the past time, so this server is far behind the current cluster’s agreement. What we expect is that the current leader will send InstallSnapshot to this server and brings it to come back to normal. But it seems to not work correctly. 🥲 I found one problem is that I forget to Unlock the Mutex Lock. 🤣
            - It seems that I can pass the PART2D Test in a high probability when reducing iterations times down to 10, which is set in the previous test.
            - getRangeEntries will lead the program into disaster in a very small probability (some corner cases). Due to some concurrent problems, it's hard to figure out what has happened.

So Everything goes well. 🖖 Hopefully, no one will be hurt by this mistake. 🥲

```bash
TestSnapshotBasic2D
ok      6.824/raft      3.201s
TestSnapshotInstall2D
ok      6.824/raft      34.506s
TestSnapshotInstallUnreliable2D
ok      6.824/raft      36.091s
TestSnapshotInstallCrash2D
ok      6.824/raft      13.126s
TestSnapshotInstallUnCrash2D
ok      6.824/raft      14.324s
```
