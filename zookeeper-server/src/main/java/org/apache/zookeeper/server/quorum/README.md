# ZAB 1.0 代码逻辑
//写规约和看代码的时候细节太多,经常看过一遍后就忘记,此MD做简要记录  
refer:  {  
协议逻辑(包括FLE+ZAB)均在quorum目录下: [quorum/](https://github.com/BinyuHuang-nju/zookeeper-build/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/server/quorum)   
部分外部逻辑如txn, zkDB等在server目录下： [server/](https://github.com/BinyuHuang-nju/zookeeper-build/tree/master/zookeeper-server/src/main/java/org/apache/zookeeper/server)   
}  
Note: leader和follower日志形式  
|      | leader                | follower | 
| ---- | --------------------- | -------- | 
| SYNC        | outstandingProposals, toBeApplied, lastProposed | packetsNotCommitted, packetsCommitted,lastQueued   | 
| BROADCAST   | outstandingProposals, toBeApplied, lastProposed    | pendingTxns, committedRequests   |  


0.	follower:**findLeader+connectToLeader** (Follower)  
	leader:创建一个线程**LearnerHandler.run()**
	* INFO:
		* leader和follower初始zabState为DISCOVERY
		
1.	follower: send **FOLLOWERINFO**   
	(Learner.registerWithLeader(FOLLOWERINFO) <- Follower.followLeader())  
	* INFO:  
		*  zxid: <acceptedEpoch, 0>  
		*  细节: sid, oa, version(0x10000)...

2.	leader: wait for quorum of **FOLLOWERINFO**  
	(Leader.getEpochToPropose(sid, lastAcceptedEpoch) <- LearnerHandler.run())  
	* INFO:
		* connectingFollowers: HashSet型,记录收到FOLLOWERINFO包的sid
		*  waitingForNewEpoch: boolean型,初值true;当self.getId()在connectingFollowers中且其满足多数派,由true变为false
		*  epoch: 整型,初值-1;根据lastAcceptedEpoch更新,为max{lastAcceptedEpoch集}  + 1
	* 等待包括自己的quorum执行getEpochToPropose，直至成功所有线程跳出该函数，或因超时抛出InterruptedException异常。
 
3.	leader: broadcast **LEADERINFO**
	(LearnerHandler L536)
	* INFO:
		* zxid: <newEpoch, 0> (newEpoch为上一步epoch终值)
		* 细节: version(0x10000)

4.	follower: process **LEADERINFO** and send **ACKEPOCH**  
	(Learner.registerWithLeader L505)  
	* process LEADERINFO:
		* newEpoch > self.acceptedEpoch: 更新acceptedEpoch,且回复ACKEPOCH
		* newEpoch = self.acceptedEpoch: 仅回复ACKEPOCH
		* newEpoch < self.acceptedEpoch: 抛出异常leader epoch更小
	* ACKEPOCH INFO:
		* lastLoggedZxid <- self.getLastLoggedZxid() <- dataTree.lastProcessedZxid  (该变量仅在processTxn被修改,故应是被apply到数据库中的最新zxid (DataTree L1105))  
		* EpochBytes: 若newEpoch > aE, := currentEpoch; 若newEpoch = aE, := -1.
	*完成后,return zxid(<newEpoch,0>), zabState := SYNCHRONIZATION.

5.	leader: wait for quorum of **ACKEPOCH**
	(Leader.waitForEpochAck(sid, ss) <- LearnerHandler.run() L540)  
	* INFO:
		* electingFollowers: HashSet型,记录收到ACKEPOCH包的sid
		* electionFinished: boolean型,初值false;当self.getId()在electingFollowers中且其满足多数派,由false变为true
		* ss与leaderStateSummary: 各自的<currentEpoch, lastProcessedZxid>
	* 在该函数中,若ss.getCurrentEpoch() != -1,会将两者ss作比较,若follower的ss比leader的ss更recent,则抛出IOException异常。
	* 误区1: lastZxid是**lastProcessedZxid**,而非lastProposedZxid,因而**FLE**中应该也是lastProcessedZxid，否则很容易出现异常。
	* 该函数同getEpochToPropose,会各线程等待直至满足推出条件,否则可能超时抛出异常。
	* 完成waitForEpochAck后,更新currentEpoch := aE, zabState := SYNCHRONIZATION.

6.	leader: syncFollower (当前不考虑SNAP情况)  
	(LearnerHandler.syncFollower(peerLastZxid, learnerMaster) <- LearnerHandler.run() L551)
	* INFO:
		* peerLastZxid: 来自ACKEPOCH的zxid,即follower端db的lastProcessedZxid  
		* minCommittedLog, maxCommittedLog: leader本地db的committedLog的最小zxid和最大zxid,运行期间仅在ZKDatabase.addCommittedProposal(request)函数中修改.作为规约,我们可以默认minCommittedLog为history头,maxCommittedLog为commitIndex.
		* lastProcessedZxid: leader自己db的lastProcessedZxid
		* 其他细节,因为(peerLastZxid < minCommittedLog)规约中永不存在,近看前几个分支可知needSnap必为false;needOpPacket,currentZxid等。
	* 处理1:
		* lastProcessedZxid = peerLastZxid: 仅DIFF, needOpPacket:=false
		* peerLastZxid > maxCommittedLog: TRUNC(maxCommittedLog),currentZxid := maxCommittedLog, needOpPacket:=false
		* minCommittedLog <= peerLastZxid <= maxCommittedLog (最常见):  
		 queueCommittedProposals(peerLastZxid, maxCommittedLog)
	* 处理2: startForwarding(lastSeenZxid) (Leader L1314)  
		* INFO: 
			* toBeApplied: 即已commit且请求被apply的提议,在TryToCommit中队尾添加元素
			* outstandingProposals：即尚未被commit的提议,在propose中队尾添加元素,在TryToCommit中去除队头元素
			* lastProposed: 即history队尾,每执行一次propose更新一次
		* 做法:
			* 1.若lastSeenZxid < lastProposed,根据toBeApplied发送PROPOSAL和COMMIT,根据outstandingProposals发送PROPOSAL.
			* 2.addForwardingFollower,即forwardingFollowers.add(这个learnerHandler).

7.	follower: syncWithLeader_1 (当前不考虑SNAP情况)  
	(Learner.syncWithLeader(<newEpoch,0>) <- Follower L109)  
	* INFO: 
		* packetsCommitted, 类似广播阶段的toBeApplied
		* packetsNotCommitted, 类似广播阶段的outstandingProposals
		* lastQueued, 类似广播阶段的lastProposed
		* snapshotNeeded, DIFF为false, TRUNC为true
		* writeToTxnLog, := !snapshotNeeded,即DIFF为true,TRUNC为false
	* 1.先接收并处理DIFF/TRUNC/SNAP:
		* DIFF: syncMode := DIFF
		* TRUNC: syncMode := TRUNC, 对zkDB做truncateLog, lastProcessedZxid := 包中的zxid
	* 2.再收到NEWLEADER/UPTODATE前循环处理包p:
		* PROPOSAL: 若(p.zxid != lastQueued+1),warn;更新lastQueued,更新packetsNotCommitted加入p
		* COMMIT: 取出packetsNotCommitted队首pif,
			* 若writeToTxnLog = false:
				-  若pif.zxid != p.zxid,warn;
				-  否则pif.zxid = p.zxid,则zk.processTxn(pif)且packetsNotCommitted.remove()去掉队首。
			* 若writeToTxnLog = true: packetsCommitted.add(p.zxid).


8.	leader: send **NEWLEADER**  
	(LearnerHandler.run() L595) 
	* 在完成某follower对应线程的syncFollower后发送
	* INFO:
		* newLeaderZxid: <newEpoch, 0> 

9.	follower: process **NEWLEADER** and send **ACK**(规约中**ACK-LD**)  
	(case Leader.NEWLEADER <- Learner.syncWithLeader L735)  
	* process NEWLEADER:
		* currentEpoch := newEpoch,即aE
		* writeToTxnLog := true,很重要,因为该follower已在forwardingFollowers中,在接收到NEWLEADER后也可能会收到PROPOSAL/COMMIT等
		* syncMode := NONE
		* startupWithoutServing(),如开启RequestProcessors线程、SessionTracker线程、注册JMX等预备工作
		* 对packetsNotCommitted中每个proposal,调用fzk.logRequest,且将其清空
			* 说明1:writeToTxnLog已为true, 故收到COMMIT与packetsNotCommitted无关
			* 说明2:logRequest最主要的是pendingTxns.add(proposal)
	* ACK INFO:
		* newLeaderZxid,即<newEpoch,0>

10.	leader: wait for quorum of **ACK-LD**  
	(Leader.waitForNewLeaderAck(sid, zxid) <- LearnerHandler.run() L627)  
	* INFO:
		* newLeaderProposal: 表达已收到ACK-LD的sid集合
		* quorumFormed: boolean型, 初值为false,当newLeaderProposal满足多数派,置为true
	* 该函数同getEpochToPropose、waitForEpochAck,会各线程等待直至满足推出条件,否则可能超时抛出异常。
	* 若zxid != 其他的<newEpoch,0>会报错。

11.	leader: broadcast **UPTODATE**