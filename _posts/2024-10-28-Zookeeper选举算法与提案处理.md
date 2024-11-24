---
title: Zookeeper选举算法与提案处理概览
categories: [ 编程, Zookeeper ]
tags: [ zookeeper ]
---

## 共识算法(Consensus Algorithm)
共识算法即在分布式系统中节点达成共识的算法，提高系统在分布式环境下的容错性。
依据系统对故障组件的容错能力可分为：
- 崩溃容错协议(Crash Fault Tolerant, CFT) : 无恶意行为，如进程崩溃，只要失败的quorum不过半即可正常提供服务
- 拜占庭容错协议(Byzantine Fault Tolerant, BFT): 有恶意行为，只要恶意的quorum不过1/3即可正常提供服务

分布式环境下节点之间是没有一个全局时钟和同频时钟，故在分布式系统中的通信天然是异步的。
而在异步环境下是没有共识算法能够完全保证一致性（极端场景下会出现不一致，通常是不会出现）

> In a fully asynchronous message-passing distributed system, in which at least one process may have a crash failure, 
> it has been proven in the famous 1985 FLP impossibility result by Fischer, Lynch and Paterson that a deterministic algorithm for achieving consensus is impossible.

另外网络是否可以未经许可直接加入新节点也是共识算法考虑的一方面，
未经许可即可加入的网络环境会存在女巫攻击(Sybil attack)

分布式系统中，根据共识形成的形式可分为
- Voting-based Consensus Algorithms: Practical Byzantine Fault Tolerance、HotStuff、Paxos、 Raft、 ZAB ...
- Proof-based Consensus Algorithms: Proof-of-Work、Proof-of-Stake ...

## Zookeeper模型与架构

> A Distributed Coordination Service for Distributed Applications: 
> ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

Zookeeper(后简称zk)定位于分布式环境下的元数据管理，而不是数据库，zk中数据存在内存中，所以不适合存储大量数据。
zk以形如linux文件系统的树形层级结构管理数据，如下图所示：
![](/assets/2024/10/28/path.png)

每一个节点称为一个`znode`除了存放用户数据（一般为1KB以内）还包含变更版本、变更时间、ACL信息等统计数据(Stat Structure)：

- `czxid`: The zxid of the change that caused this znode to be created.
- `mzxid`: The zxid of the change that last modified this znode.
- `pzxid`: The zxid of the change that last modified children of this znode.
- `ctime`: The time in milliseconds from epoch when this znode was created.
- `mtime`: The time in milliseconds from epoch when this znode was last modified.
- `version`: The number of changes to the data of this znode.
- `cversion`: The number of changes to the children of this znode.
- `aversion`: The number of changes to the ACL of this znode.
- `ephemeralOwner`: The session id of the owner of this znode if the znode is an ephemeral node. If it is not an ephemeral node, it will be zero.
- `dataLength`: The length of the data field of this znode.
- `numChildren`: The number of children of this znode.

同时znode节点可设置为以下特性：
- ephemeral: 和session生命周期相同
- sequential: 顺序节点，比如创建顺序节点/a/b，则会生成/a/b0000000001 ，再次创建/a/b，则会生成/a/b0000000002
- container: 容器节点，用于存放其他节点的节点，子节点无则它也无了，监听container节点需要考虑节点不存在的情况


Zookeeper集群中节点分为三个角色：
- Leader：它负责 发起并维护与各 Follower 及 Observer 间的心跳。所有的写操作必须要通过 Leader 完成再由 Leader 将写操作广播给其它服务器。一个 Zookeeper 集群同一时间只会有一个实际工作的 Leader。
- Follower：它会响应 Leader 的心跳。Follower 可直接处理并返回客户端的读请求，同时会将写请求转发给 Leader 处理，并且负责在 Leader 处理写请求时对请求进行投票。一个 Zookeeper 集群可能同时存在多个 Follower。
- Observer：角色与 Follower 类似，但是无投票权。

为了保证事务的顺序一致性，ZooKeeper 采用了递增的zxid来标识事务，zxid(64bit)由epoch(32bit)+counter(32bit)组成，如果counter溢出会强制重新选主，开启新纪元，如果epoch满了呢？

读操作可以在任意一台zk集群节点中进行，包括watch操作也是，但写操作需要集中转发给Leader节点进行串行化执行保证一致性。
Leader 服务会为每一个 Follower 服务器分配一个单独的队列，然后将事务 Proposal 依次放入队列中，并根据 FIFO(先进先出) 的策略进行消息发送。Follower 服务在接收到 Proposal 后，会将其以事务日志的形式写入本地磁盘中，并在写入成功后反馈给 Leader 一个 Ack 响应。
当 Leader 接收到超过半数 Follower 的 Ack 响应后，就会广播一个 Commit 消息给所有的 Follower 以通知其进行事务提交，之后 Leader 自身也会完成对事务的提交。而每一个 Follower 则在接收到 Commit 消息后，完成事务的提交。
这个流程和二阶段提交很像，只是ZAB没有二阶段的回滚操作。

## ZAB(Zookeeper Atomic Broadcast)

### Fast Leader Election(FLE)

- myid: 即sid，有cfg配置文件配置，每个节点之间不重复
- logicalclock: 即electionEpoch，选举逻辑时钟

```java

public class FastLeaderElection implements Election {
      
      // ...

      public Vote lookForLeader() throws InterruptedException {
        self.start_fle = Time.currentElapsedTime();
        try {
            /*
             * The votes from the current leader election are stored in recvset. In other words, a vote v is in recvset
             * if v.electionEpoch == logicalclock. The current participant uses recvset to deduce on whether a majority
             * of participants has voted for it.
             */
            // 投票箱, sid : Vote
            Map<Long, Vote> recvset = new HashMap<>();

            /*
             * The votes from previous leader elections, as well as the votes from the current leader election are
             * stored in outofelection. Note that notifications in a LOOKING state are not stored in outofelection.
             * Only FOLLOWING or LEADING notifications are stored in outofelection. The current participant could use
             * outofelection to learn which participant is the leader if it arrives late (i.e., higher logicalclock than
             * the electionEpoch of the received notifications) in a leader election.
             */
            // 存放上一轮投票和这一轮投票 & (FOLLOWING/LEADING)状态的peer的投票
            // 如果有(FOLLOWING/LEADING)的投票来迟了（即已经选出了leader但是当前节点接收ack的notification迟了），
            // 可根据outofelection来判断leader是否被quorum ack了，是则跟随该leader
            Map<Long, Vote> outofelection = new HashMap<>();


            int notTimeout = minNotificationInterval;

            synchronized (this) {
                // 更新当前electionEpoch
                logicalclock.incrementAndGet();
                // 更新投票为自己
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            // 投当前的leader一票，即投自己一票
            sendNotifications();

            SyncedLearnerTracker voteSet = null;

            /*
             * Loop in which we exchange notifications until we find a leader
             */
            // 循环直到有leader出现
            while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
                /*
                 * Remove next notification from queue, times out after 2 times
                 * the termination time
                 */
                // 拉取其他peer的投票
                Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

                /*
                 * Sends more notifications if haven't received enough.
                 * Otherwise processes new notification.
                 */
                if (n == null) { // 没收到peer的投票信息
                    if (manager.haveDelivered()) { // 上面的notification都发完了
                        sendNotifications(); // 再发一次
                    } else {
                        manager.connectAll(); // 建立连接
                    }

                    /*
                     * 指数退避
                     */
                    notTimeout = Math.min(notTimeout << 1, maxNotificationInterval);

                    /*
                     * When a leader failure happens on a master, the backup will be supposed to receive the honour from
                     * Oracle and become a leader, but the honour is likely to be delay. We do a re-check once timeout happens
                     *
                     * The leader election algorithm does not provide the ability of electing a leader from a single instance
                     * which is in a configuration of 2 instances.
                     * */
                    if (self.getQuorumVerifier() instanceof QuorumOracleMaj
                            && self.getQuorumVerifier().revalidateVoteset(voteSet, notTimeout != minNotificationInterval)) {
                        setPeerState(proposedLeader, voteSet);
                        Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }

                } else if (validVoter(n.sid) && validVoter(n.leader)) { // 收到其他peer的投票
                    /*
                     * Only proceed if the vote comes from a replica in the current or next
                     * voting view for a replica in the current or next voting view.
                     */
                    // 判断发送投票的peer当前状态
                    switch (n.state) {
                    case LOOKING: // 选主中
                        if (getInitLastLoggedZxid() == -1) {
                            break;
                        }
                        if (n.zxid == -1) {
                            break;
                        }

                        if (n.electionEpoch > logicalclock.get()) { // peer的electionEpoch大于当前节点的electionEpoch
                            logicalclock.set(n.electionEpoch); // 直接快进到大的epoch，即peer的electionEpoch
                            recvset.clear(); // 并清空投票箱
                            // 投票pk
                            // peer投票和当前节点进行投票pk: peerEpoch -> zxid -> myid(sid)
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                //  peer赢了，把当前节点的leader zxid peerEpoch设置为peer的
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                            } else {
                                // 当前节点赢了，恢复自己的配置
                                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                            }
                            // 将上面更新后的自己的投票信息广播出去
                            sendNotifications();
                        } else if (n.electionEpoch < logicalclock.get()) {
                            // 如果peer的electionEpoch比当前节点的electionEpoch小，则直接忽略
                            break;
                        } else
                            // electionEpoch相等，进行投票pk: peerEpoch -> zxid -> myid(sid)
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                // pk是peer赢了，跟随peer投票，并广播出去
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                                sendNotifications();
                            }

                        // 将peer的投票放入投票箱
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                        voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));

                        if (voteSet.hasAllQuorums()) {

                            // Verify if there is any change in the proposed leader
                            while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                                if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                    recvqueue.put(n);
                                    break;
                                }
                            }

                            /*
                             * This predicate is true once we don't read any new
                             * relevant message from the reception queue
                             */
                            if (n == null) {
                                setPeerState(proposedLeader, voteSet);
                                Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }

                        break;
                    case OBSERVING: // peer是观察者，不参与投票直接返回
                        LOG.debug("Notification from observer: {}", n.sid);
                        break;

                        /*
                        * In ZOOKEEPER-3922, we separate the behaviors of FOLLOWING and LEADING.
                        * To avoid the duplication of codes, we create a method called followingBehavior which was used to
                        * shared by FOLLOWING and LEADING. This method returns a Vote. When the returned Vote is null, it follows
                        * the original idea to break switch statement; otherwise, a valid returned Vote indicates, a leader
                        * is generated.
                        *
                        * The reason why we need to separate these behaviors is to make the algorithm runnable for 2-node
                        * setting. An extra condition for generating leader is needed. Due to the majority rule, only when
                        * there is a majority in the voteset, a leader will be generated. However, in a configuration of 2 nodes,
                        * the number to achieve the majority remains 2, which means a recovered node cannot generate a leader which is
                        * the existed leader. Therefore, we need the Oracle to kick in this situation. In a two-node configuration, the Oracle
                        * only grants the permission to maintain the progress to one node. The oracle either grants the permission to the
                        * remained node and makes it a new leader when there is a faulty machine, which is the case to maintain the progress.
                        * Otherwise, the oracle does not grant the permission to the remained node, which further causes a service down.
                        *
                        * In the former case, when a failed server recovers and participate in the leader election, it would not locate a
                        * new leader because there does not exist a majority in the voteset. It fails on the containAllQuorum() infinitely due to
                        * two facts. First one is the fact that it does do not have a majority in the voteset. The other fact is the fact that
                        * the oracle would not give the permission since the oracle already gave the permission to the existed leader, the healthy machine.
                        * Logically, when the oracle replies with negative, it implies the existed leader which is LEADING notification comes from is a valid leader.
                        * To threat this negative replies as a permission to generate the leader is the purpose to separate these two behaviors.
                        *
                        *
                        * */
                    case FOLLOWING: // peer正在following leader

                        Vote resultFN = receivedFollowingNotification(recvset, outofelection, voteSet, n);
                        if (resultFN == null) {
                            break;
                        } else {
                            // 成功选主，返回
                            return resultFN;
                        }
                    case LEADING: // peer 是 leader
                        /*
                        * In leadingBehavior(), it performs followingBehvior() first. When followingBehavior() returns
                        * a null pointer, ask Oracle whether to follow this leader.
                        * */
                        Vote resultLN = receivedLeadingNotification(recvset, outofelection, voteSet, n);
                        if (resultLN == null) {
                            break;
                        } else {
                            return resultLN;
                        }
                    default:
                        break;
                    }
                } else {
                    // 推举的leader或投票的peer不合法，直接忽略
                    // ...
                }
            }
            return null;
        } finally {
            // ...
        }
    }


    private Vote receivedFollowingNotification(Map<Long, Vote> recvset, Map<Long, Vote> outofelection,
            SyncedLearnerTracker voteSet, Notification n) {
        /*
         * Consider all notifications from the same epoch
         * together.
         */
        if (n.electionEpoch == logicalclock.get()) { // 同一轮投票
            //            若对方选票中的electionEpoch等于当前的logicalclock，
            //            说明选举结果已经出来了，将它们放入recvset。
            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
            voteSet = getVoteTracker(recvset, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
            // 判断quorum是否满足选主条件
            if (voteSet.hasAllQuorums() &&
                    // 判断推举的leader已经被quorum ack了，避免leader挂了导致集群一直在选举中
                    checkLeader(recvset, n.leader, n.electionEpoch)) {
                // leader是自己，将自己设置为LEADING，否则是FOLLOWING(或OBSERVING)
                setPeerState(n.leader, voteSet);
                Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                // 清空消费投票的queue
                leaveInstance(endVote);
                return endVote;
            }
        }

        // 到这里是
        // peer的electionEpoch和logicalclock不一致
        // 因为peer是FOLLOWING，所以在它的electionEpoch里已经选主成功了

        /*
         * 在跟随peer选出的leader前，校验这个leader合法不合法
         * Before joining an established ensemble, verify that
         * a majority are following the same leader.
         *
         * Note that the outofelection map also stores votes from the current leader election.
         * See ZOOKEEPER-1732 for more information.
         */
        outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
        voteSet = getVoteTracker(outofelection, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

        if (voteSet.hasAllQuorums() && checkLeader(outofelection, n.leader, n.electionEpoch)) {
            synchronized (this) {
                logicalclock.set(n.electionEpoch);
                setPeerState(n.leader, voteSet);
            }
            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
            leaveInstance(endVote);
            return endVote;
        }

        return null;
    }

    private Vote receivedLeadingNotification(Map<Long, Vote> recvset, Map<Long, Vote> outofelection, SyncedLearnerTracker voteSet, Notification n) {
        /*
        * 在两个节点的集群中（leader+follower），如果follower挂了，recovery之后，因为投票无法过半（follower会首先投自己一票），会找不到leader
        * In a two-node configuration, a recovery nodes cannot locate a leader because of the lack of the majority in the voteset.
        * Therefore, it is the time for Oracle to take place as a tight breaker.
        * */
        Vote result = receivedFollowingNotification(recvset, outofelection, voteSet, n);
        if (result == null) {
            /*
            * Ask Oracle to see if it is okay to follow this leader.
            * We don't need the CheckLeader() because itself cannot be a leader candidate
            * */
            // needOracle，当集群无follower & 集群voter==2 时，
            if (self.getQuorumVerifier().getNeedOracle() 
                    // 且cfg配置中key=oraclePath的文件（默认没有，askOracle默认false）中的值 != '1' 时会走到if里
                    // 这里可参考官网 https://zookeeper.apache.org/doc/current/zookeeperOracleQuorums.html
                    && !self.getQuorumVerifier().askOracle()) {
                LOG.info("Oracle indicates to follow");
                setPeerState(n.leader, voteSet);
                Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                leaveInstance(endVote);
                return endVote;
            } else {
                LOG.info("Oracle indicates not to follow");
                return null;
            }
        } else {
            return result;
        }
    }
    
    // ...
}
```


### 提案处理


![](/assets/2024/10/28/zkServer.png)

所有的提案均通过leader来提，follower接受的提案会转发到leader。
zk采用责任链模式对请求进行处理，不同的角色（leader/follower/observer）对应不同的责任链：

![](/assets/2024/10/28/processorChain.png)

以下是leader的各个Processor的作用

- `LeaderRequestProcessor`: Responsible for performing local session upgrade. Only request submitted directly to the leader should go through this processor.
- `PrepRequestProcessor`: It sets up any transactions associated with requests that change the state of the system
- `ProposalRequestProcessor`: 调用`Leader#propose`将proposal加入发送给follower的queue，由LeaderHandler异步发送给follower和处理follower的ack
- `SyncRequestProcessor`: 将request写磁盘 
- `AckRequestProcessor`: ack leader自己的request
- `CommitProcessor`: 提交提案。`CommitProcessor`本身是一个线程，上游调用先把request加入队列，然后异步消费处理
- `ToBeAppliedRequestProcessor`: simply maintains the toBeApplied list
- `FinalRequestProcessor`: This Request processor actually applies any transaction associated with a request and services any queries


接收follower的ack并提交走下面的调用：
```java
org.apache.zookeeper.server.quorum.Leader.LearnerCnxAcceptor#run
org.apache.zookeeper.server.quorum.Leader.LearnerCnxAcceptor.LearnerCnxAcceptorHandler#run
org.apache.zookeeper.server.quorum.Leader.LearnerCnxAcceptor.LearnerCnxAcceptorHandler#acceptConnections
org.apache.zookeeper.server.quorum.LearnerHandler#run
org.apache.jute.BinaryInputArchive#readRecord
org.apache.zookeeper.server.quorum.LearnerMaster#processAck 这里如果满足quorum则调用CommitProcessor
org.apache.zookeeper.server.quorum.Leader#tryToCommit
org.apache.zookeeper.server.quorum.Leader#commit (leader 发送commit消息给follower，此时leader还不一定提交了，因为异步处理的) 
org.apache.zookeeper.server.quorum.Leader#inform (leader 发送inform消息给observer，此时leader还不一定提交了，因为异步处理的)
```

判断是否满足quorum的方法为：`SyncedLearnerTracker#hasAllQuorums` ，

```java
public class SyncedLearnerTracker { // Proposal的父类，即每个提案一个Tracker

    public static class QuorumVerifierAcksetPair {
        private final QuorumVerifier qv; // 每一个zxid就是一个QuorumVerifier
        private final HashSet<Long> ackset; // ack的sid set
        ...
    }
  
    protected ArrayList<QuorumVerifierAcksetPair> qvAcksetPairs = new ArrayList<>();
    ...

    public boolean hasAllQuorums() {
        for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
            if (!qvAckset.getQuorumVerifier().containsQuorum(qvAckset.getAckset())) {
                return false;
            }
        }
        return true;
    }
   
    ...
}
```

最终调用`QuorumMaj#containsQuorum`：

```java
public class QuorumMaj implements QuorumVerifier {
    ...
    protected int half = votingMembers.size() / 2;
    
    /**
     * Verifies if a set is a majority. Assumes that ackSet contains acks only
     * from votingMembers
     */
    public boolean containsQuorum(Set<Long> ackSet) {
        return (ackSet.size() > half);
    }
    ...
```


## 参考
- [Consensus Algorithms in Distributed Systems](https://www.baeldung.com/cs/consensus-algorithms-distributed-systems)
- [FLP Impossibility Result](https://www.the-paper-trail.org/post/2008-08-13-a-brief-tour-of-flp-impossibility/)
- [zookeeperInternals](https://zookeeper.apache.org/doc/r3.4.13/zookeeperInternals.html#sc_atomicBroadcast)
- [详解分布式协调服务 ZooKeeper，再也不怕面试问这个了](https://mp.weixin.qq.com/s/DwyPt5YZgqE0O0HYEC1ZMQ)
- [Lecture 8: Zookeeper](http://nil.csail.mit.edu/6.824/2020/video/8.html)
- [ZooKeeper: Wait-free coordination for Internet-scale systems](http://nil.csail.mit.edu/6.824/2020/papers/zookeeper.pdf)
- [diff_acceptepoch_currentepoch](https://qisiii.github.io/post/tech/distributed/zookeeper/diff_acceptepoch_currentepoch/)
- [Zookeeper(FastLeaderElection选主流程详解)](https://blog.csdn.net/wangshouhan/article/details/89919404)
- [zookeeper-framwork-design-message-processor-leader](https://donaldhan.github.io/zookeeper/2021/01/05/zookeeper-framwork-design-message-processor-leader.html)

