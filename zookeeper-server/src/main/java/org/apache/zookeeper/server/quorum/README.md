# ZAB 1.0 代码逻辑
//写规约和看代码的时候细节太多,经常看过一遍后就忘记,此MD做简要记录  
refer:  {  
协议逻辑(包括FLE+ZAB)均在quorum目录下: [quorum/](https://github.com/BinyuHuang-nju/zookeeper-build/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/server/quorum)   
部分外部逻辑如txn, zkDB等在server目录下： [server/](https://github.com/BinyuHuang-nju/zookeeper-build/tree/master/zookeeper-server/src/main/java/org/apache/zookeeper/server)   
}

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
		* newEpoch = self.acceptedEpoch: 回复ACKEPOCH
		* newEpoch < self.acceptedEpoch: 抛出异常leader epoch更小
	* ACKEPOCH INFO:
		* lastLoggedZxid <- self.getLastLoggedZxid() <- dataTree.lastProcessedZxid  (该变量仅在processTxn被修改,故应是被apply到数据库中的最新zxid (DataTree L1105))  
		* EpochBytes: 若newEpoch > aE, := currentEpoch; 若newEpoch = aE, := -1.

5.	leader: wait for quorum of **ACKEPOCH**
	(Leader.waitForEpochAck(sid, ss) <- LearnerHandler.run() L540)  
	* INFO:
		* electingFollowers: HashSet型,记录收到ACKEPOCH包的sid
		* electionFinished: boolean型,初值false;当self.getId()在electingFollowers中且其满足多数派,由false变为true
		* ss与leaderStateSummary: 各自的<currentEpoch, lastProcessedZxid>
	* 误区1: lastZxid是lastProcessedZxid,而非lastProposedZxid
	* 在该函数中,若ss.getCurrentEpoch() != -1,会将两者ss作比较,若follower的ss比leader的ss更recent,则抛出IOException异常。
	* 该函数同getEpochToPropose,会各线程等待直至满足推出条件,否则可能超时抛出异常。

6.	leader: syncFollower