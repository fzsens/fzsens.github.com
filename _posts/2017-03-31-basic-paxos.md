---
layout: post
title: Basic Paxos理解
date: 2017-03-31
categories: consensus
tags: [paxos]
description: 对基础Paxos算法的解读和推导
---

>Paxos算法的描述和证明都非常简洁，但是核心的概念都非常抽象，如果需要工程化应用于实际的系统中需要完成的工作更多。最近对`Paxos`的研究，参考了很多互联网上的资料，这篇文章试图对这些参考做一个记录，并提供一个适于我的角度理解的`Paxos`算法解读。

### Paxos解决的问题

如我们在上一篇文章[《多副本和分布式一致性》](http://fzsens.github.io/consensus/2017/03/29/replications-and-consensus/ "多副本和分布式一致性")中讨论的，在分布式系统中一个核心的问题：“在多个不可靠的节点中，达成一个确定的状态”。而这也正是`Paxos`算法解决的问题。

网上对`Paxos`算法的讨论已经很多，大多数文章都会在开头吐槽一下`Paxos`论文，也就是《 Paxos Made Simple》、《The Part-Time Parliament》的难懂，在我看来这些“难懂”更多是因为算法本身过于抽象，`Leslie Lamport`对于算法本身的细节了然于胸，但是对其中的很多隐喻、以及可能存在的不同场景，却很缺少直观的描述。以致于可能全文都已经看完，却还是不知道`Paxos`算法为什么可以保证分布式的一致性、以及我们怎么在系统中应用。

### 线性推导

>这是在知乎上一个知友对[Paxos的推导过程](https://www.zhihu.com/question/19787937/answer/82340987)，和[《多副本和分布式一致性》](http://qiongsong.com/consensus/2017/03/29/replications-and-consensus/ "多副本和分布式一致性")类似，但是使用更加直白的描述性语言，对于最初理解Paxos算法很有帮助，下文为对整个推导过程的复述，我尽量做到没有隐含条件，尽量详细通俗地表述整个过程。

开始推导之前，先完成本次推导中一致的初步定义：

	假设存在N个进程集合：P1~PN，每一个进程中都存在一个值v，我们记为P1.v ~ PN.v。每一个进程都可以对自己进程以及其他进程的值发出修改信息，同时也会接收自己和其他进程的修改消息：我们定义达成这个N进程一致性的原则是：N个进程中，有超过一半的进程（多数派）认为v的值是相同的，并且确定达成一致后的v值是无法进行更改的。
	
	为什么不是所有的进程的v值都达成一致，而是选择其中的多数派达成一致？主要是基于异常（可用性）的考虑，每一个进程在工作的过程中，都可能会出现“故障-恢复-故障”的情况，如果严格要求所有的进程都一致，显然无法达到可用性的要求；因此退而求其次，我们认为只要足够多的进程认为v值是一致的也是唯一，那么重新加入的进程就可以获取这一个唯一确定的v值，最终实现所有进程都一致的要求；而足够多的进程，目的就是确定P1~PN进程集合中已经确定唯一的值v。于是我们定义这个足够多为 Q-qrm = N/2 + 1，既多数派的定义。显而易见，一个集合的存在多个多数派，但是每两个多数派之间必然存在交集，因此只要一个达成一致性要求的多数派 Q 确定值 v = a，并且值不可变，那么可以得出当前集合中达成一致性要求的任意多数派 Q' 确定的v也必然是a。

	举一个例子简单说明：集合{A、B、C、D、E}，假设存在一个满足一致性定义的多数派 Q = {A、B、E}，根据定义，存在：A.v = B.v = E.v = a；如果产生了另一个满足一致性定义的多数派 Q'= {C、D、E}，则存在C.v = D.v = E.v，由于每一个进程确定的v值不可变，因此可以推导出C.v = D.v = E.v = a；
	
	因此在我们定义的一致性协议中，允许部分节点出现故障，主要多数进程可以工作，整个系统就可以工作。这个多数进程的数量是一个绝对值，比如P1-P5，那么这个值就是3，允许任意2个进程故障，但是不允许3个以及3个以上的进程出现故障。

在我们下面的推导中，为了简化文字描述，我们定义进程集合{P1、P2、P3}，后续会定义一系列的约束和规则来保证P1、P2、P3的值，无论在什么条件下，最后都可以达成一致。集合中的进程之间是平等的，不依赖于特定的进程；进程之间的通信只能通过消息来完成，并且不存在消息被篡改的问题。

推导过程中，我们使用线性推导的方式，既先假设一个场景，协议流程可以保证一致性，然后再变换场景，使得当前的协议流程无法保证一致性，再调整协议流程，不断优化最终得到目标协议。

场景1：P1 => v=a; P2 => v=b，并且都已经完成了自身v值的设置，然后同时向P3发送各自的v值。

![](/assets/img/paxos/paxos-basic-situation1.png)

很明显，最终的一致性值取决于P3接受(Accept)了那个值，站在P3的角度，首先v不可能既是a又是b，而且只要接受其中的一个之后，就形成了多数派，无法再修改v值。因此我们先使**用先来后到**的策略，接收先送达的值，拒绝后达的值，于是存在下面两种情况：

1. P1先送达 -> Q-qrm = a，并拒绝后到的P2;
2. P2先送达 -> Q-qrm = b，并拒绝后到的P1;

因此在场景1种，我们的协议为：

	S1: 如果之前没有收到任何信息，那必须接受第一次接收到的值。

>因为进程之间平等，如果能无条件拒绝接受，则可能没有任何值会被接受，不满足活性（Liveness：是那些最终一定会发生的事情）要求。

场景2：P1 => v=a; P2 => b; P3 => c，并且都完成了自身值的设置。

![](/assets/img/paxos/paxos-basic-situation2.png)

这种情况下，显然永远无法形成多数派一致。为了保证活性，我们需要加强协议，首先对于场景2，如果想最终达到一致性，需要增加

	S2.1: 如果进程的v不是最终的确定值，那么进程能够多次修改自己的v值

接着继续思考，既然必须修改值才能达成一致，那每一个进程就必须从接受到的消息中选择接受和拒绝的消息？单纯先来后到的策略显然无法满足，因此我们为消息引入ID策略，根据ID策略，来区分各个消息。ID为一个自然数，每一个消息的ID都互不相同，这样可以根据消息ID的大小来选择接受或者拒绝，这里我们不妨选择
	
	S2.2: 进程接受消息ID更大的值，而拒绝消息ID更小的值。

因为每个消息各不相同，假设有消息ID关系：P1.id > P2.id > P3.id，对于场景2，可能出现

1. P1的消息早于P2的消息到达：根据策略，我们会接受P1 => v=a; 拒绝随后送达的P2; 最终多数派一致值为v=a
2. P2的消息早于P1的消息到达：根据策略，我们会接受P2 => v=b，并且此时已经达成了多数派一致值为v=b；随后P1到达，因为P1.id > P2.id，因此P3也没有理由拒绝P1 => v=a，此时形成的多数派一致值v=a，和只有一个确定值的定义矛盾；

显然我们目前的协议还无法处理这种场景。

为了方便后面的描述，我们对术语进行进一步的明确定义，进程P发送试图修改其他进程值的消息称为提案 Proposal；提案的ID为 proposal_id；如果提案中附带的值被决定为是v的最终值，即大多数接受者接受了这个提案，那么称提案被通过；进程P记录接受(accept)的提案ID为 a_proposal_id，a_proposal_id的定义为当前接受的提案ID。假设进程接收(receive)到 Proposal-i之前已经接受(accept)了多个Proposal，我们把了a_propsal_id定义为已经接收的 Proposal中 proposal_id最大的哪一个。在接收 Proposal-i的时候，修改 a_proposal_id = Max( a_proposal_id, Porposal-i.proposal_id)；

进程P同时承担着两个功能

	1. 进程主动尝试修改并决定v的值，先确定自己的v值，并向进程集群中的其他进程广播提案，称为 **Proposer**
	2. 进程被动接受其他进程广播的提案，并确定是决绝(reject)还是接受(accept)提案，称为 **Acceptor**

接着考虑，对于**场景2-2**出现的P2消息早于P1到达的情况，我们首先考虑有几种处理方法
	
	(1). P3拒绝P2
	(2). P3拒绝P1
	(3). 限制P1的值等于P2的值，P3先接受P2，然后接受P1

对于方法(1)，在P3没有任何前置条件的情况下明显无法拒绝(S1和S2.2保证)。P3需要获得更多的信息，因为进程之间只能通过消息进行通信，因此如果需要拒绝P2，存在以下两种可能
	
	1. P2事先发出声明消息，让P3可以拒绝P2，这显然是不可能的，如果P2事先知道自己会被拒绝，那P2只需要不发出消息即可
	2. P1事先发出声明消息，让P3可以拒绝P2，在目前的协议下，如果P1能够在P2的Proposal到达P3之前，将P1.proposal_id通知P3，并将P3.a_proposal_id设置为P1.proposal_id，即可让P3拒绝P2

对于方法(2)，目前的协议中拒绝一个Proposal，都是基于proposal_id来实现的，在不引入其他约束的情况下，P3只能接受P1的提案，暂时将这个方法排除在外。

对于方法(3)，因为我们最终关心的并不是具体哪一个进程提出的提案被确定，而是被确定的提案的值是否能够满足一致性要求。假设能够保证P1.proposal.v = P2.proposal.v，那么在接受(accept)P2之后，也可以接受(accept)P1，并且不会带来一致性问题。这个假设要求P1能够在发送提案之前就得到P2的提案值，在进程只能通过消息通信的前提下，存在下面两种选择

	1. P1等待P2发送消息通知P2.proposal的值
	2. P1主动询问P2.proposal的值

我们先从方法(1)入手

1. 方法(1)中的声明消息，显然不同于Proposal，但是又需要包括接下来可能要发出的Proposal，因此我们把这个声明消息称为预提案 Preproposal，预提案中带有接下来要提交的提案的proposal_id；

2. 当作为Acceptor的进程Pj收到预提案preproposal-i之后，需要对自己的Pj.a_proposal_id更新，存在两种处理方案
	1. 如果preproposal-i.proposal_id < Pj.a_proposal_id，因为proposal_id都是递增的，很明显，这个preproposal之后又提案ID更大的提案被提出，而根据(S2.2协议内容)这个preproposal对应的proposal注定会被拒绝，因此Pj选择忽略这个预提案，或者返回一个错误信息；
	2. 如果preproposal-i.proposal_id > Pj.a_proposal_id，则接受这个preproposal，并将Pj.a_proposal_id设置为preproposal-i.proposal_id

增加预提案之后，P1和P2都会先发出各自的preproposal之后再发出对应的proposal。让我们看看引入了预提案之后，能解决**场景2-2**中出现的问题，

	场景2中，
	因为每个消息各不相同，假设有消息ID关系：P1.id > P2.id > P3.id，对于场景2，可能出现
	1. P1的消息早于P2的消息到达：根据策略，我们会接受P1 => v=a; 拒绝随后送达的P2; 最终多数派一致值为v=a
	2. P2的消息早于P1的消息到达：根据策略，我们会接受P2 => v=b，并且此时已经达成了多数派一致值为v=b；随后P1到达，因为P1.id > P2.id，因此P3也没有理由拒绝P1 => v=a，此时形成的多数派一致值v=a，和只有一个确定值的定义矛盾；

1. 如果P1.preproposal早于P2.preproposal或者P2.proposal到达P3，则P3会顺利拒绝P2的预提案或者提案，协议能最终达成一致，并且 v=a；

2. 如果出现：P2.preproposal和P2.proposal都先于P1.preproposal和P1.proposal到达的情况，这种情况和原来的场景情况完全一致，协议中**依然存在矛盾**

既：单纯引入预提案还是无法保证消息在乱序到达时候，可能带来的不一致。在**场景2-2**中体现为：预提案无法保证P2的预提案或者提案会被P1的预提案或者提案阻止接受，从而使P2达成一致性；而P2的预提案和提案又无法阻止P1达成一致性。

回顾目前我们的协议中存在两个角色（Acceptor和Proposer），两种消息（Proposal和PreProposal），一个进程完成提案需要经过两个阶段

预提案阶段：

1. Proposer：向各个Acceptor广播预提案，预提案中附带接下来要提交的提案ID proposal.proposal_id
2. Acceptor：接收到预提案，从预提案中获取对应的提案ID proposal_id，和自己的a_proposal_id进行比较，如果a_proposal_id > proposal_id，则忽略或者返回错误；如果a_proposal_id < proposal_id，则将 a_proposal_id设置为 proposal_id

提案阶段：

1. Proposer：使用之前预提案中的proposal_id作为Proposal.proposal_id向Acceptor广播提案Proposal
2. Acceptor：接收到提案Proposal，判断是否满足a_proposal_id > Proposal.proposal_id 如果是，则拒绝这个提案，否则跟新 a_proposal_id为 proposal_id，并接受Proposal中的v值

我们目前的场景假设都是基于3个进程的集合进行讨论的，其实很容易推广到N个进程，如果P1代表Pi，P2代表Pj，一个v被确定，代表有一个进程的提案被多数派集合接受，因此存在下面的情况：

1. Pi被通过，则需要有多数派Q-i都接受了Pi的提案
2. Pj被通过，则需要有多数派Q-j都接受了Pj的提案

Q-i和Q-j之间必然存在不为空的交集 Q-i&j，从中选择一个进程 Pk，因为 Q-i&j中所有的提案值是一致的，因此Pk就决定了整个进程集合N的一致值是Pi.v(Pi.proposal.v)还是Pj.v(Pj.proposal.v)，类比到之前的场景，Pk就是P3。

为了不失一般性，下面的讨论基于N个进程的讨论，使用Pi和Pj代表P1和P2，作为任意进程，Pk代表P3代表影响选择Pi或者Pj的关键进程。并且 Pi.proposal_id(Pi.proposal.proposal_id) > Pj.proposal_id(Pj.proposal.proposal_id)；Pi.v = a; Pj.v = b。

针对**场景2-2**继续我们的讨论，这个场景我们的拒绝策略时效，并带来不一致结果的原因是因为Pj已经被Pk接受(accept)实际已达成一致性状态，但是Pk却无法拒绝Pi，接受Pi又会改变一致性状态确定的值。而方法(2)暂时被排除在外，因此我们需要从方法(3)入手，通过限制Pi的值，使得Pk可以既接受Pj又接受Pi，同时又不破坏一致性状态确定的值。

让我们再一次确定**场景2-2**的前提：

1. Pj.proposal提案被确定，既Pj.preproposal和Pj.proposal都被Pk接受，并且存在多数派进程集合Q-j接受了Pj.proposal

2. Pi.proposal_id > Pj.proposal_id

在方法(2)，Pi需要主动询问进程集，确定得到Pj.v = b。因为进程之间平等，显然Pi并不知道Pj到底是哪一个进程，因此只能通过消息在进程集中广播来获取，并从其他进程的回复中获取Pj的提案值。于是问题转换为需要收到多少个进程的回复才能够确定这些回复中包括了Pj.proposal。答案也是多数派，记Pi收到的多数派回复为Q-i，因为Q-j的存在，因此Q-i ∩ Q-j ≠ NULL，可知Q-i中必然存在Pm接受过Pj.proposal。

于是现在站在Pi角度的协议流程变化为：

1. Pi向其他各个进程发出询问请求，并等待多数派回复，从回复值中获取Pj.v
2. Pi向其他各个进程发出预提案，如果多数派回复接受，则进入提案阶段
3. Pi使用预提案的proposal_id向其他各个进程发出提案。

课件步骤1和步骤2没有以来关系，并且完成的事情是一样的，都是向其他各个进程发出消息，等待多数派回复。因此将查询步骤和预提案步骤合并为预提案步骤。

即为Proposer也就是 Pi向其他各个进程发出预提案(假设为全集N)，等待多数派回复(Q-i)，从回复中获取Pj.v，并进入提案阶段；Acceptor也就是 Q-i(这里为了方便表述，选择Pm作为代表)则接收(receive)预提案preproposal，如果可以接受(accept)这个预提案preproposal既 Pi.preproposal，将自己接受(accept)过的提案proposal也就是Pj.proposal，作为返回值回复给询问进程 Pi。

但是实际上这个时候 Pm可能不只接受到Pi和Pj的preproposal和proposal，甚至不知道最后Pj.proposal最后会不会被通过。为了保险起见，Pm选择回复所有曾经接受(accept)过的proposal，这样虽然效率较低，但是保证不会遗漏。

Q-i集合中的进程都和Pm一样，回复了它们所接受过的提案，把这部分提案记为K-i。Pi会从K-i中选择一个Proposal Pc作为Pi.proposal的值，既Pi.proposal.v = Pc.proposal.v。在**场景2-2**中，Pj.proposal已经被确定，因此我们的目标提案是Pj.proposal，只要我们能保证Pc.proposal.v = Pj.proposal.v即可。换而言之，Q-i中可能存在 Pf.proposal.v ≠ Pj.proposal.v，Pi的任务就是避免从中选择到 Pf.proposal。

假设我们已经拥有策略CL，Pi通过CL可以从Q-i中选择 Pc，使得 Pc.proposal.v = Pj.proposal.v

先我们观察一下 Pf.proposal的特征，Pf能够被接受的前提是得到 Q-f集合进程在预提案阶段通过的保证，既通过Pf.preproposal。Q-f和Q-j必然有存在交集 Ps, Ps既收到并通过Pf.preproposal又通过了Pj.proposal。对于Ps而言这两个消息的到达顺序可能为：

1. Ps先收到Pf.preproposal
2. Ps先收到Pj.proposal

而Pf.preproposal.proposal_id和Pj.proposal.proposal_id之间也存在两种可能：Pf.preproposal.proposal_id > Pj.proposal.proposal_id，或者 Pf.preproposal.proposal_id < Pj.proposal.proposal_id，因此存在4种组合情况。

先假设 Pf.preproposal.proposal_id > Pj.proposal.proposal_id

则情况1：Ps先收到Pf.preproposal后，会将ps.a_proposal_id设置为Pf.preproposal.proposal_id，因此Pj.proposal不可能被选中，于这个场景的假设矛盾；

情况2：Ps先收到Pj.proposal，因为CF策略的存在，因此Ps会将Pj.proposal返回给 Pf，使得 Pf.proposal.v = Pj.proposal.v，这与 Pf.proposal.v ≠ Pj.proposal.v矛盾。

因此 Pf.preproposal.proposal_id < Pj.proposal.proposal_id 必然成立，根据提案和预提案的proposal_id关系，也有Pf.preproposal.proposal_id = Pf.proposal.proposal_id < Pj.proposal.proposal_id。

因此加入策略CL存在，则在K-i中的回复提案中，存在Pf.proposal.v ≠ Pj.proposal.v，必然有 Pf.proposal.proposal_id < Pj.proposal.proposal_id。**而所有的proposal_id大于Pj.proposal_id的提案如果它能够被提出，其值都等于Pj.proposal_id.v**，这就是CF的具体形式。又因为 Pj.proposal存在 K-i中，也就是从K-i中挑选出proposal_id最大的proposal，就可以保证其值必然会等于Pj.proposal.v。

我们得策略CF存在是建立在CF存在的基础之上。反过来这个CL策略的具体形式，能不能推论出下面的结论，还需要进一步论证

	CP1：如果一个提案Pj.proposal最终会被通过，那么对于任意的一个提案Pi.proposal,如果Pi.proposal.proposal_id > Pj.proposal.proposal_id，那么Pi.proposal.v = Pj.proposal.v

将CF策略纳入我们的协议中，然后从协议流程中推论出CP1即可完成整个协议流程的推导，目前我们的协议流程为：

阶段一：Proposer向所有的Acceptor广播预提案Preproposal；Acceptor收到Preproposal之后，对比Preproposal.proposal_id和自己的a_proposal_id，如果Preproposal.proposal_id > a_proposal_id，则将a_proposal_id修改为Preproposal.proposal_id，并将自己之前所有接受(accept)过的所有提案proposal，作为返回值回复给Proposer，如果未接受过提案则返回null；如果Preproposal.proposal_id < a_proposal_id，则直接忽略请求

阶段二：Proposer接受到多数派进程的回复，从回复中选择提案ID最大的提案中的值，作为自己提交Proposal的值，如果为null，则任意选择一个值，然后向所有的Acceptor广播自己的提案，和阶段一的Preproposal共用相同的proposal_id；Acceptor收到提案之后，判断proposal.proposal_id >= a_proposal_id，如果成立则接受提案值，并更新a_proposal_id = proposal.proposal_id，否则直接忽略请求。

> 实际上null值也可以当作是一个普通的值

使用反证法来完成证明过程。假设存在 Pi.proposal.proposal_id > Pj.proposal.proposal_id，并且 Pi.proposal.v ≠ Pj.proposal.v 则

1. Pi.proposal能够被提出，必然存在多数派进程集合Q-i在阶段一通过Pi.preproposal，并给出回复集合K-i，选择K-i中的proposal_id最大的proposal记为MaxProposal(K-i).proposal，并且有Pi.proposal.v = MaxProposal(K-i).proposal.v ≠ Pj.proposal.v

2. Pj.proposal被确定必然存在多数派进程集合Q-j，Q-j中的进程都接受(accept)了Pj.proposal

3. Q-i和Q-j至少存在交集合 Pk，Pk既接受(accept)了Pi的预提案 Pi.preproposal又接受了 Pj的提案 Pj.proposal

4. Pk接受(accept) Pi.preproposal和 Pj.proposal的顺序可能为
	1. Pk先接受到 Pi.preproposal，然后接受到 Pj.proposal
	2. Pk先接受到 Pj.proposal，然后接受到 Pi.preproposal
由于 Pi.proposal.proposal_id > Pj.proposal.proposal_id, 如果是情况1，Pk.a_proposal_id >= Pi.preproposal.proposal_id = Pi.proposal.proposal_id > Pj.proposal.proposal_id，Pk必然会拒绝Pj.proposal。因此只能是情况2，也就是Pj.preproposal送达Pk的时候，Pk已经接受(accept)了Pj.proposal。因此Pk回复给Pi的提案集合中必然包括了提案Pj.proposal。

5. 因此K-i中一定存在一个Pm，使得Pm.proposal.v = MaxProposal(K-i) ≠ Pj.proposal.v，根据策略CL，也必然有Pm.proposal.proposal_id > Pj.proposal.proposal_id。

到这边我们可以推论出，Pi.proposal.proposal_id > Pj.proposal.proposal_id，并且 Pi.proposal.v ≠ Pj.proposal.v 成立，必然存在Pm.proposal.proposal_id > Pj.proposal.proposal_id，并且由于每一个提案ID都是是唯一性的自然数，必然存在Pi.proposal.proposal_id > Pm.proposal.proposal_id > Pj.proposal.proposal_id。假设Pi.proposal.proposal_id = Pj.proposal.proposal_id + 1，则显然不可能存在自然数 Pm.proposal.proposal_id 能够满足定义。因此CP1得证。

CP1约束了，如果一个Pj.proposal被确定，则不可能存在Pi.proposal.proposal_id > Pj.proposal.proposal_id，并且 Pi.proposal.v ≠ Pj.proposal.v，同理可得，如果Pj.proposal被通过，那么必然不存在Po被确定，并且 Po.proposal.proposal_id < Pj.proposal.proposal_id，并且 Po.proposal.v ≠ Pj.proposal.v，将CP1引用在Po上即可推出。

再优化一下协议流程中的阶段一，Acceptor在接受(accept)，并返回提案值的时候，将所有之前接受过的proposal都返回给了询问的Proposer。而Proposer在最终选择MaxProposal(K-i)的时候实际上是选择最大值，因此Acceptor只需要返回曾经接受过的proposal中proposal_id最大的proposal即可。最终的协议流程为：

阶段一：Proposer向所有的Acceptor广播预提案Preproposal；Acceptor收到Preproposal之后，对比Preproposal.proposal_id和自己的a_proposal_id，如果Preproposal.proposal_id > a_proposal_id，则将a_proposal_id修改为Preproposal.proposal_id，并将自己之前所有**接受(accept)过的所有提案proposal中proposal_id最大的proposal**，作为返回值回复给Proposer，如果未接受过提案则返回null；如果Preproposal.proposal_id < a_proposal_id，则直接忽略请求

阶段二：Proposer接受到多数派进程的回复，从回复中选择提案ID最大的提案中的值，作为自己提交Proposal的值，如果为null，则任意选择一个值，然后向所有的Acceptor广播自己的提案，和阶段一的Preproposal共用相同的proposal_id；Acceptor收到提案之后，判断proposal.proposal_id >= a_proposal_id，如果成立则接受提案值，并更新a_proposal_id = proposal.proposal_id，否则直接忽略请求。

这就是`Paxos算法`的协议流程。

### 原论文解读和参考资料

对于`Paxos算法`论文的的证明和说明，网上的资料浩如烟海，写得也都非常详细，我就不再赘述，只把一些要点做归纳总结

* [Paxos Made Simple译文](http://dsdoc.net/paxosmadesimple/index.html)

注：《Paxos Made Simple》的论文原文阅读并不困难，推荐这篇译文主要可以结合最后的译者注，对照自己对文章的理解，其中也补全了部分原文中没有说明的隐喻。

* [Paxos算法1-算法形成理论](http://blog.csdn.net/chen77716/article/details/6166675#reply)

注：

1. 这篇文章是目前我读过对Paxos算法介绍最为细致的中文资料，特别是对推论过程中的逐步分析，可见作者的功力。
2. 其中对于P2c的推论出P2的部分，需要多看几遍，和原文一样，这是一个从抽象反推出具体的过程，使用的是反向的推理。而非一般的从具体到抽象的归纳和总结。
3. 上一个章节的线性推理中，则是先假设了这样一个P2c过程的存在，通过存在来进行证明。最后论证P2c能满足P2，两个对比起来看可能会容易理解一些。
4. 使用的数学归纳法，归纳法第一步：m成立 => m+1成立，可以推广到 [m+1,n-1)成立。第二步在证明[m,n-1)成立的基础上，推论出n的成立。使用还是多数派必然存在交集的原理。
5. P1并不完备，既P1存在无法解决的矛盾-接受了第一个提案之后，如何处理后续收到的提案；P2c在推论出P2的时候，同时通过协议流程对接受和拒绝的描述，补全P1完备性。

* [Paxos算法2-算法过程](http://blog.csdn.net/chen77716/article/details/6170235#reply)

注：

1. 这一篇和上一篇属于同一个作者的同一个系列，通过对Paxos算法过程的讨论，引出实现中可能会遇到的问题
2. 其中关于Paxos和Master-Slave/Master-Master的说明，实际上Paxos算法并不需要一个Leader/Master，选择Leader的考虑是为了减少可能出现的活锁(多个proposer在prepare阶段相互覆盖proposal_id)，就算是出现多个Leader，Paxos算法也可以保证一致性。而M-S需要Master是一个固定的角色，[《多副本和分布式一致性》](http://qiongsong.com/consensus/2017/03/29/replications-and-consensus/ "多副本和分布式一致性")有对应的描述，可以对比一下。
3. 关于Paxos和两阶段提交，Paxos的两个阶段和2PC有异曲同工之妙，每一个单独的Proposer和Acceptor的交互都可以看作是一次2PC，但是Paxos重点突破的是2PC的单点问题，2PC如果作为接入点的协调者出现了异常整个应用是不可用的。而Paxos，每一个进程都是平等的，都可以承担接入点的角色，可以在(N-1)/2个节点出现故障的时候保证一致性。
4. 关于Paxos和3PC的区别，文章中没有描述，实际上3PC是对2PC的一个改进，通过引入协调者选举、超时和precommit作为中间状态来解决2PC的单点故障和堵塞问题。但是3PC可能会引起网络分区问题，假设一个网络被分割成为两个彼此无法进行通信的网络，A部分都受到了precommit，B部分都没有，可能会选择出两个协调者，A部分的协调者决定提交precommit，而B分别的协调者没有提交这个precommit，造成数据不一致。也有改进的3PC，也是通过多数派原则来尝试解决3PC的问题[《Increasing the Resilience of Distributed and Replicated Database Systems》](http://people.csail.mit.edu/idish/ftp/JCSS.pdf)。

* [Paxos理论介绍(1): 朴素Paxos算法理论推导与证明](https://zhuanlan.zhihu.com/p/21438357?refer=lynncui)

注：这是微信的PhxPaxos组件的开发者介绍Basic Paxos的文章，可以通过给出的表格来加深对选举过程的理解。

* [Paxos算法3-实现探讨 ](http://blog.csdn.net/chen77716/article/details/6172392)

注：这部分，重点讨论了Paxos实现时候，主要角色需要完成的功能可微信的[《微信自研生产级paxos类库PhxPaxos实现原理介绍》](http://mp.weixin.qq.com/s?__biz=MzI4NDMyNTU2Mw==&mid=2247483695&idx=1&sn=91ea422913fc62579e020e941d1d059e#rd)可以互为补充。

### 总结

以上为最近对Basic Paxos算法的理解，Paxos算法的核心原理简单，主要是两个`Majority`之间必然存在交集，通过这个交集可以在多个2PC之间建立联系从而达成一致性。但是从核心原理推导出具体的协议策略却需要较长的推导。从Basic Paxos到具体能够在生产环境中工程化使用又有更多需要优化和处理的异常。但是作为目前分布式一致性理论的代表作，无疑是值得花时间去研究和理解的。