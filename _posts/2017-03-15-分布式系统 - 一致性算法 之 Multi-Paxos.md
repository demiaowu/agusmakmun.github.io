---
layout: post
title:  "分布式系统 - 一致性算法 之 Multi-Paxos"
date:   2018-03-15 23:18:10 +0700
categories: [分布式,算法]
---

* Kramdown table of content
{:toc, toc}

## 算法描述 
当前multi-paxos没有权威的文献参考，目前大多采用的参考文献[1]的描述，这里首先给出其翻译版，然后分析其各个模块的细节，Lead election，CommitIndex Advance，Log Replication等。

### 服务器状态  
**所有acceptor上面持久化的状态**

| **状态**           | **解释**                                                |
|--------------------|---------------------------------------------------------|
| lastLogIndex       | 已经accept的最大的log entry index                       |
| minProposal        | 最小的Proposal Id                                       |
| firstUnchosenIndex | acceptedProposal[index] !=”无穷大”的最小log entry index |

每个acceptor上面存放了一个复制日志，日志index (1 \<= index \<=
lastLogIndex)。每个Log Entry包含如下内容：

| 状态                | 解释                                            |
|---------------------|-------------------------------------------------|
| acceptedProposal[i] | 这个log entry最后接受的Proposal Id，初始化时为0 |
| acceptedValue[i]    | 这个log entry最后接受的value，初始化为null      |

**所有proposer上面持久化的状态**

| 状态     | 解释                           |
|----------|--------------------------------|
| maxRound | Proposer见过的最大round number |

**在Proposer里经常改变的**

| 状态      | 解释                                                                                                                                                                                                                        |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| nextIndex | 下一个可用的log entry的index                                                                                                                                                                                                |
| prepared  | 如果 prepared 为true，那么proposer就不需要进行prepare阶段， 直接进行accept阶段，否则需要prepare阶段。prepared初始化为 false，当收到大多数acceptor的prepare reqeust的相应消息 noMoreAccepted为true时，会将prepared设置为true |

### **Prepare RPC** (Phase 1)

**Request**:

| 参数       | 解释                          |
|------------|-------------------------------|
| proposalId | 当前提案的proposalId          |
| index      | 用户当前提案的log entry index |

**Response:**

| 参数               | 解释                                                                                         |
|--------------------|----------------------------------------------------------------------------------------------|
| acceptedProposalId | Acceptor接受的acceptedProposal [index]                                                       |
| acceptedValue      | Acceptor接受的acceptedValue[index]                                                           |
| noMoreAccepted     | 如果这个acceptor在\>=index的log entry中，没有接受任何的value， 则设置为true，否则设置为false |

接收者实现：

当接收到prepare request后，如果request. proposalId \>= minProposal,
则设置minProposal = request. proposalId。并且保证拒绝所有的minProposal \<
request. proposalId的accept请求。

### **Accpet RPC** (Phase 2)

**Request**:

| 参数        | 解释                                                                                                                        |
|-------------|-----------------------------------------------------------------------------------------------------------------------------|
| ProposalId  | 当前提案的ProposalId                                                                                                        |
| index       | 用户当前提案的log entry index                                                                                               |
| value       | 一个提案的value，如果prepare阶段受到了reponse，那么就是termId最大的那提案的Value，否则proposer设置自己的Value（来自Client） |
| commitIndex | 发送者的commitIndex                                                                                                         |

**Response:**

| 参数        | 解释                  |
|-------------|-----------------------|
| minProposal | Acceptor的minProposal |
| commitIndex | Acceptor的commitIndex |

**接收者实现**：

如果proposalId \>= minProposalId就执行如下设置：

acceptedProposal[index] = request. proposalId;

acceptedValue[index]=request.value;

minProposal=request.proposalId;

对于每一个index \< request.commitIndex的log entry，如果其acceptedProposal
[index] == n，则可以设置其acceptedProposal [index] = “无穷大”

### **Success** (Phase 3)

**Request**:

| 参数  | 解释                  |
|-------|-----------------------|
| index | 提案的log entry index |
| value | 提案的value           |

**Response:**

| 参数        | 解释                  |
|-------------|-----------------------|
| commitIndex | Acceptor的commitIndex |

**接收者实现**：

设置

acceptedProposal [index] = “无穷大”

acceptedValue[index] = request.value;

**发送者实现**：

if reply.commitIndex \< commitIndex，则发送Success(index = reply.commitIndex +
1; value = acceptedValue[reply.commitIndex+1])

### **Proposer Algorithm**   
 
Write(inputValue) -\> return bool

1: 如果不是leader、或者Leader还没有初始化完成，直接返回false；

2: 如果prepared 为true  

&nbsp;&nbsp;&nbsp;&nbsp; (a)Let index = nextIndex, nextIndex++.  

&nbsp;&nbsp;&nbsp;&nbsp; (b) Go to step 6.  

3: 设置 index = firstUnchosenIndex and nextIndex = index + 1.  

4: maxRound ++; 并生成一个新proposal number赋值给n  

5: 广播Prepare(n; index) 请求给所有的acceptors  

6: 当从大多数acceptor收到Prepare Response (reply:acceptedProposal, reply:acceptedValue, reply:noMoreAccepted)  

&nbsp;&nbsp;&nbsp;&nbsp; 如果最大的reply.acceptedProposal不等于0，那么使用它的reply.acceptedValue，否则使用自己的inputValue;  

&nbsp;&nbsp;&nbsp;&nbsp; 如果所有的acceptor在返回的都是reply:noMoreAccepted = true,那么设置prepared = true;  

7: 向所有的acceptor广播Accept(index; n; v) 请求  

8: 当收到一个Accept response(reply:n, reply:firstUnchosenIndex)时。  

&nbsp;&nbsp;&nbsp;&nbsp; 如果reply.n \> n，则从reply.n中设置maxRound，并设置prepared = false. 跳转到Step 1.  

&nbsp;&nbsp;&nbsp;&nbsp; 如果reply.firstUnchosenIndex ≤ lastLogIndex and acceptedProposal[reply:firstUnchosenIndex] = “无穷大”
那么就发送Success(index = reply:firstUnchosenIndex; value = acceptedValue[reply:firstUnchosenIndex])。  

9: 当接收到大多数acceptor的accept response，且reply.n = n, 那么设置acceptedProposal[index] = “无穷大”，并且acceptedValue[index] = v.  

10: 如果v == inputValue, return true.  

11: Go to step 2.  


## 算法分析

### Leader election
在Multi-paxos中，只有两种角色，proposer和acceptor，其中leader既是proposer，也是acceptor，其他的服务节点都是acceptor。因为现在没有Leader，那么也就是没有proposer，不能发起提案。

### Log replication

### CommitIndex advance

### Crash recovery

### Membership change

## 参考文献
[1]. paxos-summay. 

----------------

