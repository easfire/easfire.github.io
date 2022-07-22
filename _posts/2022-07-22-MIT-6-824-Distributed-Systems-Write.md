![v2-f10d28fc1d6014b86e8820efcf3f2f01_1440w](https://user-images.githubusercontent.com/24760084/180398813-c15ea31b-cf18-43e7-8b4e-7726fd6b1ebc.png)

## 6.824 分布式系统（Google File System[GFS] ② Write） [MIT公开课]

## Write 概述
the writes are more complex and interesting. the application interface for writes is pretty similar, there's just some library call to mate you make to the GFS client library saying look here's a file name and a range of bytes, I'd like to write and the buffer of data that I'd like you to write to that range actually, let me backpedal I only want to talk about record append and so I'm going to praise this, the client interface as the client makes a library call that says here's a file name and I'd like to append this buffer of bytes to the file, I said this is the record append that the paper talks about.

写入更复杂和有趣。

写入的应用程序接口非常相似，只是有一些库调用来配合你，对 GFS 客户端库说看这里是一个文件名和一个字节范围，我想把数据缓冲区的数据写到这个文件的这个范围内，让我退后一步（收敛简化一下场景），我只想讨论数据的追加写。喜欢将这个字节缓冲区追加到文件中，我说这是论文谈到的记录追加。

client wants to append this file and asks the master where to look for the last chunk in the file, because the client may not know how long the file is, if lots of clients are opinion to the same file because we have some big file this logging stuff from a lot of different clients may be you know no client will necessarily know how long the file is and therefore which offset or which chunk it should be appending to so you can ask the master please tell me about the server's that hold the last current chunk in this file.

客户想要追加这个文件，并询问master 在哪里寻找文件中的最后一个块，因为客户可能不知道文件有多长。如果很多客户对同一个文件追加写，可能没有一个客户端一定会知道文件有多长，因为他们没法知道其他客户端写了多少，也就不知道该往什么偏移量或者往哪个chunk追加数据。因此它应该附加到哪个偏移量或哪个块，所以你可以问 master，请告诉我关于服务器的保存的此文件中的最后一个当前chunk。

if you're reading you can read from any up-to-date replica, but for writing though these needs to be a primary so at this point on the file, may or may not have a primary already designated by the master, so we need to consider the case of if there's no primary already and all the master knows well there's no primary.

如果你正在Read data，你可以从任何最新的副本中读取。但是对于写入需要是主副本，所以在文件的这一点上，可能有也可能没有被master已经指定的主副本，所以我们 需要考虑是否已经没有主副本，并且所有master 都知道没有主副本的情况。

one case is no primary. in that case the master needs to find out the set of chunk servers that have the most up-to-data copy of the chunk, because if you've been running the system for a long time, due to failures or whatever there may be chunk servers out there that have old copies of the chunk from yesterday or last week that I've been kept up to data, because maybe that server was dead for a couple days and wasn't receiving updates, so there's you need to be able to tell the difference between up-to-date copies of the chunk and non up-to-date.

一种情况是 没有主副本。 在这种情况下，master 需要找出具有该chunk 的**最新**数据副本的一组块服务器(包含primary和secondary replica)，因为如果您已经运行系统很长时间，由于故障或任何可能的原因，那里有块的旧副本的块服务器，比如从昨天或上周开始断更的，尽管我一直在更新数据。因为可能该服务器已经宕机了几天并且没有接收更新，所以你需要能够区分块的最新副本和非最新副本之间的区别。

## No primary on master (Write)

### Find up-to-date replicas
the first step is to find up-to-data this is all happening in the master because the client has asked the master which chunk server to talk if I want to append this file. so a part of the master trying to figure out which chunk severs the client should talk to. when we finally find up-to-date replicas and means is a replica whose version of the chunk is equal to the version number that master knows is the most up-to-date version number, it's the master that hands out these version numbers, the master remembers that oh for this particular chunk, the trunk server is only up-to-date if it has version number 17 and this is why it has to be non-volatile stored on disk, because if it was lost in a crash, and there were chunk servers holding stale copies of chunks the master wouldn't be able to distinguish between chunk servers holding stale copies of a chunk from last week and a chunk server that holds the copy of the chunk that was up-to-date as of the crash, that's why the master members of version number on disk.

第一步是查找最新数据，这一切都发生在master 中，客户端如果要附加此文件，master 的一部分会试图弄清楚客户端应该与哪台块服务器交谈。当我们最终找到最新的副本时，这意味着一个块副本的版本号等于 master 知道的最新版本号的版本号。要知道是 master 分发的这些版本，也就只有master可以做这个判断。例如，对于这个特定的块，chunk server 只有在版本号为 17 时才是最新的，这就是为什么它必须是非易失性存储在磁盘上的原因。因为，如果它在崩溃中丢失, 并且有块服务器持有过期的块副本，主服务器将无法区分，持有上周的块的陈旧副本的块服务器和持有最新块的副本但发生了崩溃的块服务器，这就是master 会在磁盘上持续更新并记录版本号。

<img width="1178" alt="image" src="https://user-images.githubusercontent.com/24760084/180398559-2b2da766-4416-4d7d-b773-fc4b819e3dc2.png">

Q&A:

Q1: 为什么不将所有chunk server上保存的最大版本号，作为master记录的某chunk的最新版本号？

if you knew you were talking to all the chunk servers, so the observation is the master has to talk to the chunk servers anyway if it reboots in order to find which chunk server holds the up-to-date version, because the master doesn't remember that, so you could just take the maximum you could just talk to the chunk servers find out which chunks and versions they hold, and take the maximum for a given chunk overall the responding chunk servers. and it would work if all the chunk servers holding a chunk responded. but the risk is that at the time the master reboots maybe some of the chunk servers are offline or disconnected or whatever themselves rebooting and don't respond and so all the master gets back is responses from chunk servers that have last week's copies of the block and the chunk servers have the current copy haven't finished rebooting or offline.

如果您知道您正在与所有块服务器进行通信，那么观察结果是，如果master 重新启动以查找哪个块服务器拥有最新版本，那么它无论如何都必须与块服务器通信，因为master不会记这个信息，所以你可以与块服务器交谈，找出它们持有哪些chunks以及它们的版本，并在响应块服务器中为给定块取版本号的最大值。 如果所有持有块的块服务器都响应，这样做是可以的。 但风险在于，在master 重新启动时，可能某些块服务器处于脱机状态或断开连接，或者任何自身重新启动而没有响应，如此一来 master得到的都是来自具有上周块副本的块服务器的响应，并且块服务器的当前副本尚未完成重新启动或脱机。

### Pick Primary & Secondaries

if the servers holding the most recent copy are permanently dead if you've lost all copies of the most recent version of a chunk.

如果保存最新副本的服务器永久死亡，如果您丢失了某个chunk的所有最新版本的拷贝。会怎么样。

so the question is the master knows that for this chunk is looking for version 17, supposing it finds no chunk server and it talks to the chunk servers periodically to sort of ask them what chunks do you have what versions you have, supposing it finds no server with version 17 for this chunk, then the master will either say well either not respond yet and wait or it will tell the client "look, I can't answer that, try again later". and this would come up like there was a power failure in the building and all the servers crashed, and we're slowly rebooting, the master might come up first and you know some fraction of the chunk servers might be up and other ones would reboot five minutes from now, but so we ask to be prepared to wait and it will wait forever, because you don't want to use a stale version of that chunk.

所以问题是 master 知道这个 chunk正在寻找版本 17，假设它没有找到这样的块服务器，并且它定期与块服务器对话，以询问他们有哪些块，它们都是什么版本，假设master 没有找到对于这个块的版本为 17 的服务器，那么master 要么不响应并等待，要么它会告诉客户端 “看，我无法回答，稍后再试。。” 这种极端情况是会出现的，就像建筑物中出现电源故障并且所有服务器都崩溃了，我们正在慢慢重新启动，master 可能会先启动，并且部分块服务器可能会启动，而其他服务器会在五分钟后重新启动，但我们要求准备好等待，它可能将永远等待，因为您不想使用该块的陈旧版本，也就是说这种情况下，最新的版本17 的chunk server 最总将无法被找到。

so the master needs to assemble the list of chunk servers that have the most recent version, the master knows the most recent versions stored on disk, each chunk server along with each chunk also remembers the version number of the chunk they stored. so when chunk servers report into the master saying look I have this chunk, the master can ignore the ones whose version does not match the version the master knows is the most recent.

所以 Master 需要组装具有最新版本的chunk server列表，master 知道存储在磁盘上的最新版本，每个chunk server 同样也会记录他们存储的chunk的版本号。 所以当块服务器向 master 报告说，看我有这个块时，Master 可以忽略 版本与master 知道的最新版本不匹配的那些chunk servers，注意这里在描述确立Primary的过程。

### Increment Version# and Tells Primary & Secondary
so remember we were the client want to append the master doesn't have a primary it figures out maybe you have to wait for the set of chunk servers that have the most recent version of that chunk it picks a primary. so I'm gonna pick one of them to be the primary and the others to be secondary servers among the replicas set at the most recent version. the master then increments the version number and writes that to disk, so it doesn't forget it the crashes and then it sends the primary in the secondaries and that's each of them a message saying "for this chunk here's the primary and here's the secondaries", you know recipient maybe one of them and here's the new version number, so then it tells primary and secondaries this information plus the version number, the primary and secondaries all write the version number to disk so they don't forget, because you know if there's a power failure or whatever they have to report in to the master with the actual version number they hold.

所以请记住，我们是客户端想要追加数据，并且在master 没有 primary的情况下，Master 发现也许你必须等待一组具有该块的最新版本的块服务器们，并从中选择出一个主服务器 (primary chunk server)。 所以我将在最新版本的副本中选择其中一个作为主要服务器，其他作为辅助服务器。 然后master **递增版本号**，并将其写入磁盘，然后Master 向所有的Primary和Secondary发送消息，说“对于这个块，谁是主服务器，谁是辅助服务器 "，Primary和Secondary都会把版本号写入磁盘，如果出现电源故障或其他任何情况，他们必须使用他们持有的实际版本号向 master 报告。

### Master writes up-to-date Version# to Disk

Q&A:

Q1: 块服务器返回了一个比master持有的版本号大的版本号？

the paper says if the master reboots and talks to chunk servers and one of the chunk servers reboot, reports a version number, that's higher than the version number the master has. the master assumes that there was a failure while it was assigning a new primary and adopts the new higher version number that it heard from a chunk server, so it must be the case that in order to handle a master crash at this point, that the master writes its own version number to disk after telling primary and secondary. I don't think it could happen.

该论文说，如果master 重新启动并与块服务器对话，并且其中一个块服务器重新启动，则会报告一个版本号，该版本号高于主服务器的版本号。 master 假设在分配一个新的主节点时发生了故障，并采用了它从块服务器听到的新的更高版本号，因此为了处理此时的 master 崩溃，必须是这样的情况，master在告诉primary和secondary之后将自己的版本号写入磁盘。 我认为这不可能发生块服务器持有比master版本号大的版本?

--- 版本号是master 在选出新primary chunk server之后，基于它之前持有的up-to-date 版本号递增得到的，并且在通知当前所有的Primary（也许此刻还没被声明为Primay）和Secondaries，刷新他们自己的版本号之前，会先把新版本号落地到master的磁盘位置，所以不管在什么情况下，master持有的某个chunk的最新版本号一定会是系统当前最新的版本号。

![image](https://user-images.githubusercontent.com/24760084/180401091-0fd81a88-f14c-45ec-b9f2-3bc726fe897c.png)

### Tells Primary, Version# and give it a Lease

the master tells the primaries and secondaries that they're allowed to modify this block it also gives the primary a lease which basically tells the primary look you're allowed to be primary for the next 60 seconds, and after 60 seconds you have to stop. this is part of the machinery for making sure that we don't end up with two primaries. I'll talk about a bit later.

master 告诉主节点和次节点他们可以修改这个块，master 还给主节点一个租约，告诉主节点你可以在接下来的 60 秒内做主节点，60 秒后你将失去这个角色任命。这是为了确保我们不会以两个primaries。 我稍后再进一步谈这个点。

<img width="1291" alt="image" src="https://user-images.githubusercontent.com/24760084/180401570-478a69fb-71db-4e20-bd4b-ff20fccb72fa.png">

### Primary Picks offset, all replicas told to write at offset

now we have primary and the master tells the client who the primary and the secondary are. the paper explains a sort of clever way to manage this in some order or another, the client sends a copy of data it wants to be appended to the primary and secondaries, and the primary and secondaries write the data to a temporary location, it's not appended to the file. the primary maybe is receiving these requests from lots of different clients concurrently it picks some order execute, the client request one at a time and for each client append request the primary looks at the offset, that's the end of the file, end of the current chunk makes sure there's enough remaining space in the chunk and then tell primary and all the secondaries also to write the data to end of chunk and the same offset.

现在我们有了Primary : ) Master会告诉客户端primary 和secondaries 是谁。 该论文解释了一种以某种顺序进行管理的**聪明方法**，客户端发送一份它想要附加到主服务器（Primary）和辅助服务器的数据副本，主服务器和辅助服务器将数据写入临时位置，它不是开始变直接附加到文件中。 主服务器可能同时从许多不同的客户端接收这些请求，它会选择一定的顺序执行，客户端一次请求一个，并且对于每个客户端追加请求，主服务器会查看偏移量，即文件的结尾，并要确保块中有足够的剩余空间，然后告诉主节点和所有辅助节点也将数据写入块的末尾和相同的偏移量。

so the primary picks an offset all the replicas including the primary are told to write the new appended record at offset. the secondaries may do it or not, either run out of space maybe they crashed maybe the network message was lost from the primary, so if a secondary actually wrote the data to its disk at that offset, it will reply yes to the primary, if the primary collects a yes answer from all of the secondaries, so if all of them managed to actually write and reply to the primary saying yes I did it. then the primary is going to reply success to the client. if the primary doesn't get an answer from one of the secondaries or the secondary reply sorry something bad happened I ran out of disk space, then the primary replies no to the client. the paper says if the client gets an error like that back in the primary the client is supposed to reissue the entire append sequence starting again talking to the master to find out the most grease the chunk at the end of the file. I want to know the client supposed to reissue the whole record append operation.

所以主节点选择一个偏移量，包括主节点在内的所有副本都被告知在偏移量处写入新的附加记录。**辅助节点**可能会这样做，或者空间不足，可能它们崩溃了，可能网络消息从主节点丢失，所以如果，辅助节点实际上在该偏移量处将数据写入了磁盘，它将向主节点回复“是”，如果primary 收到了所有secondary的“是”答复，所有secondary 都真正写成功并回复primary 说“是的，我做到了”，然后primary 将向客户端回复成功。如果primary 没有从其中一个辅助服务器得到答案，或者辅助服务器回复抱歉，发生了不好的事情，我的磁盘空间用完了，那么primary 会向客户端回复“否”。该论文说，如果客户端在primary 中遇到类似的错误，则客户端应该重新发出整个附加序列，再次开始与主服务器对话，以找出文件末尾的最大块。之后，客户端应该重新发出整个记录追加操作。

<img width="784" alt="image" src="https://user-images.githubusercontent.com/24760084/180401818-41a18fcc-c415-45a4-a718-36e337c23b22.png">

## Q&A:

- Q1: 为什么需要每个secondary都成功写入，才可以返回客户端 yes？

the primary tells all the replicas to do the append, maybe some of them do, some of them don't, but some replicas where append succeeded they did append, so now we have replicas donor the same data, but that is just the way GFS works.

主节点告诉所有副本进行追加，也许其中一些写成功，一些没有，但是一些追加成功的副本他们确实追加了，所以现在我们有副本提供相同的数据，但GFS确实强校验了这个过程，必须所有副本都写成功，才返回客户端 Yes，这就是GFS的方式。

- Q2: 写文件失败后（即客户端收到master回复的no），读chunk数据有什么不同？

if a reader then reads this file they depending on what replica they be they may either see the appended record or they may not, that means zero or more of the replicas may have appended the record of that offset and the other ones not. so what you which were roughly read from you may or may not see the record.

如果读取这个文件，要取决于他们是什么副本，他们可能会看到附加的记录，也可能不会，这意味着零个或多个副本可能已经附加了该偏移量的记录，而其他的则没有。 因此，您大致可能会或者可能不会看到想要的记录。

- Q3: 是否可以通过版本号来判断副本是否有之前追加的数据？

all the secondaries are the same version number, so the version number only changes when the master assigns a new primary which would ordinarily happen and probably only happen if the primary failed(no primary), so what we're talking about is replicas that have the fresh version number and you can't tell from looking at them that they're missing that the replicas are different but maybe they're different and the justification for this is that, maybe the replicas don't all have that the appended record, but that's the case in which the primary answer no to the client and the client knows that the write failed and the reason behind this is that then the client library will reissue the append so the appended record will show up you know eventually the append succeed you would think, because the client I'll keep reissuing it until succeeds and then when it succeeds that means there's gonna be some offset you know farther on in the file, where that record actually occurs in all the replicas as well as offsets preceding that would only occurs in a few of the replicas.

所有的辅助节点都是相同的版本号，版本号只有在master 分配一个新的primary 时才会发生变化，这通常会发生并且可能只有在 primary 故障时才会发生 (即前面讨论的no primary 的场景)，所以我们正在谈论的是具有新版本号的副本，并且您无法通过查看它们来判断它们缺少副本是不同的，但也许它们是不同的，这样做的理由是，也许副本并不都有那个附加的记录，但在这种情况下，对客户端的主要回答是“否”，并且客户端知道写入失败，其背后的原因是客户端库将重新发出附加记录，因此附加的记录将显示出来，您最终知道您会认为追加成功，因为客户端将继续重新发布这个追加的记录直到成功，然后当它成功时，这意味着您将在文件中进一步了解一些偏移量，其中该记录实际上出现在所有副本中，而之前的偏移量只会出现在少数副本中。

- Q4: 客户端将数据拷贝给多个副本会不会造成性能瓶颈？

the exact path that the right data takes might be quite important with respect to the underlying network and the paper somewhere says even though when the paper first talks about it he claims that the client sends the data to each replica in fact later on it changes the tune and says the client sends it to only the closest of the replicas and then that replica forwards the data to another replica along a sort of chained until all the replicas had the data and that path of that chain is taken to sort of minimize crossing bottleneck inter switch links in a data center.

正确数据所采用的确切路径对于底层网络可能非常重要，并且某处的论文说，即使当论文第一次谈到它时，他声称客户端将数据发送到每个副本实际上后来它改变了，调整并说客户端仅将其发送到最近的副本，然后该副本将数据沿某种链接转发到另一个副本，直到所有副本都具有数据，并且采用该链的路径以最小化交叉瓶颈在数据中心内的交换机间链路。

- Q5: 如果写数据失败了，不是应该找到问题在哪并且重试吗？

in the ordinary sequence there already be a primary for that chunk, the master sort of will remember, there's already a primary and secondary for that chunk and it won't go through this master selection it won't increment the version number, it will just tell the client look up here's the primary with no version number change.

在普通序列中，那个块已经有一个主节点，master会记住那个块已经有一个主节点和次节点，它不会再通过这个master去选举新的primary，也不会增加版本号。master只需告诉客户这里有primary，没有需要更新的版本号。

<img width="1337" alt="image" src="https://user-images.githubusercontent.com/24760084/180412452-4347625d-9c47-4818-a2a3-def0975e5d06.png">

in this scenario in which the primaries isn't answered failure to the client you might think something must be wrong with something and that it should be fixed before you proceed, in fact as far as I can tell, the paper there's no immediate anything, and then the client retries the append, you know because maybe the problem was a network message got lost so there's nothing to repair right, now we're gonna message got lost we should be transmitted and this is sort of a complicated way of retransmitting the network message maybe that's the most common kind of failure in that case just we don't change anything, it's still the same primary same secondaries the client tries maybe this time it'll work, because the network doesn't discard a message, it's an interesting question though that if what went wrong here is that one of that there was a serious error or fault in one of the secondaries what we would like is for the master to reconfigure that set of replicas to drop that secondary that's not working and it would then because it's choosing a new primary in executing this code path the master would then increment the version and then we have a new primary and new working secondaries with a new version and this not-so-great secondary with an old version and a stale copy of the data but because that has an old version the master will never mistake it for being fresh. but there's no evidence in the paper that happens immediately as far as what's said in the paper, the client just retries and hopes it works again later. eventually the master will if the secondary is dead eventually does ping all the trunk servers will realize that and will probably then change the set of primaries and secondaries and increment the version but only later.

在这种情况下，客户没有回答初选失败，您可能会认为某些东西一定有问题，应该在您继续之前修复它，但事实上据我所知，论文没有提到需要直接做什么处理，然后客户端重试追加数据。你知道，因为问题可能是网络消息丢失了，所以没有什么可以修复的。现在我们的消息丢失了，它们应该被传输，这是一种重新传输的复杂方式。在这种情况下，网络消息可能是最常见的故障，只是我们没有更改任何内容，仍然是之前的primary和secondaries，客户端再次重试，也许这次它会起作用。因为网络不会丢弃消息，这是一个有趣的问题。

如果这里出了什么问题是其中一个从属服务器中存在严重错误或故障，我们希望主服务器重新配置该组副本，以剔除这台不工作的辅助服务器，然后它会因为它在执行此代码路径时（下图所示的代码）选择一个新的primary，然后master 会增加版本，然后我们有一个新的主节点和具有新版本的新工作辅助节点，而这个不太好的辅助节点有一个旧版本和数据的陈旧副本，但因为它有旧版本，主人永远不会误认为它是新鲜的。但是就论文中所说的而言，论文中没有证据表明这个过程会立即发生，客户只是重试并希望以后再次起作用。最终，如果辅助服务器死了，主服务器最终会 ping 所有chunk 服务器，并将意识到这一点，然后可能会更改主服务器和辅助服务器的集合并增加版本，但只是在以后。

<img width="1291" alt="image" src="https://user-images.githubusercontent.com/24760084/180412706-7bea7ad6-127d-43ac-ad8a-f9f24399dbbd.png">

- Q6: the lease.

the leases that the answer to the question what if the master thinks the primary is dead, because it can't reach it, that's supposing we're in a situation where at some point the master said you're the primary and the master was like painting them all the servers periodically to see if they're alive, because if they're dead and wants to pick a new primary, the master sends some pings to you, you're the primary? and you don't respond. so you would think that at that point where gosh you're not responding to my pings then you might think the master at that point would designate a new primary. it turns out that by itself is a mistake and the reason why it's a mistake to do that simple. use that simple design is that I may be pinging you and the reason why I'm not getting responses is because then there's something wrong with a network between me and you, so there's a possibility that you're alive and you're the primary, I'm ping you the network is dropping that packets but you can talk to other clients and you're serving requests from other clients and if the master sort of designated a new primary for that chunk, now we'd have two primaries processing writes but two different copies of the data and so now we have totally diverging copies the data and that's called that error having two primaries or whatever processing requests without knowing each other. it's called split brain. and I'm writing this on board because it's an important idea and it'll come up again and it's caused or it's usually said to be caused by network partition that is some network error in which the master can't talk to the primary but the primary can talk to clients sort of partial network failure.

这就是租约。如果master 认为primary 已经死了，因为它无法到达它。那么这个问题的答案是假设我们处于这样一种情况，在某个时候master 说你是 primary，于是master 就像定期在所有服务器上绘制它们一样，以查看它们（primary）是否还活着，因为如果它们已死，并且想要选择一个新的primary，master 会向您发送一些 ping，您是primary 吗？而你没有回应（由于任何原因吧）。所以你会认为在那个时候你没有响应我的 ping，那么你可能会认为，那个时候的master 会指定一个新的 primary。事实证明，这本身就是一个错误的，也就是做那么简单的设计可能带来的错误。使用那个简单的设计是我可能正在ping你，而我没有收到回复。原因可能是因为我和你之间的网络有问题，所以有可能你还活着并且你是primary，网络丢弃了这些数据包，但恰巧您可以与其他客户端正常通信，并且您正在为其他客户端的请求提供服务。如果此时master 为这个chunk 指定了一个新的primary，那么现在我们将有两个primary在工作，并且它们拥有两个不同的数据副本 ( totally deverging copies)！所以现在我们有完全不同的数据副本，这就是所谓的致命错误，有两个primary并且它们不知道彼此的存在，却在同时处理着请求。它被称为**裂脑(split brain)**。我在黑板上写下这个，是因为这是一个重要的想法，它会再次出现，它是由网络分区引起的，或者通常说是由网络分区引起的，这是一些网络错误，其中master 无法与primary 对话，但是primary 可以与客户端通信，因为部分网络故障。

<img width="1261" alt="image" src="https://user-images.githubusercontent.com/24760084/180413462-e0acdfb3-89c7-4276-ab29-4b70dd92f818.png">

these are the hardest problems to deal with and building these kind of storage systems. so that's the problem is we want to rule out the possibility of mistakingly designating two primaries for the same trunk. the way the master achieves that is when it designates a primary, it says it gives a primary Elyse which is basically the right to be primary until a certain time the master knows how long the lease lasts and the primary knows how long is lease lasts, if the lease expires the primary knows that it expires and will simply stop executing client requests, it'll ignore or reject client requests after the lease expired and therefor if the master can't talk to the primary and the master would like to designate a new primary the master must wait for the lease to expire for the previous primary so that means master is going to sit on its hands for one lease period 60 seconds, after that it's guaranteed the old primary will stop operating its primary and now the master can see if he does need a new primary without producing this terrible split brain situation.

这些是构建这类存储系统最难处理的问题。所以问题是我们要排除掉 这种错误地为同一个chunk 指定两个primary的可能性。Master 实现这一点的方式是，当它指定一个Primary时，它说它赋予了一个Primary Elyse，这基本上是Primary被授权了，直到某个时间，Master 知道租约持续多长时间，Primary 知道租约持续多长时间，如果租约到期，Primary 知道它已经到期 并且将停止响应客户端的请求，它会在租约到期后忽略或拒绝客户端的请求。因此如果Master 无法与Primary 对话并且Master 希望指定一个新的Primary，那么必须等待前一个Primary 的租约到期，这就意味着，Master 将在 这60 秒过程中，坐等一个租约过期，从而保证旧Primary已经停止执行Primary的职责，现在Master 可以再看看它是否确实需要一个新的Primary，而不会产生这种可怕的脑裂情况。

- Q7: why designated a new primary is bad since the clients always ask the master first and so the master changes its mind then subsequent clients will direct the clients to the new primary?

为什么指定一个新的主节点是不好的? 因为客户端总是先询问Master，之后Master 改变主意，后续客户端会将客户端引导到新的Primary。--- 也就是说，让不能应答的旧Primary在最长60秒任期内自生自灭，最终还是只有一个新Primary在提供服务。这是有问题的。

one reason is that the clients cache for efficiency, the clients cache the identity of the primary for at least for short periods of time, even if they didn't though the bad sequence is that, for example I'm the master, you ask me who the primary is, I send you a message saying the primary is server_1 and that message is inflate in the network, and then I'm the master I think somebody's failed whatever I think that primary is failed, I designated a new primary and I send the primary message saying you're the primary and I start answering other clients who ask the primary is, and saying that over there is the primary while the message to you is still in flight you receive the message saying the old primary the primary, you think gosh I just got this from the master I'm gonna go talk to that primary and without some much more clever scheme there's no way you could realize that even though you just got this information from the master, it's already out of date and if that primary servers your modification requests now we have to and respond success to you right. then we have two conflicting replicas.

一个原因是客户端缓存是为了提高效率，客户端至少会在短时间内缓存主服务器的身份，即使他们没有缓存。糟糕的时序情况还是会发生，具体是这样的，例如我是Master，你Client 问我谁是Primary，我给你发一条消息说Primary是 server_1 并且该消息在网络中传播。然后作为Master的我探测发现，这个Primary出现故障，我立刻指定了一个新的Primary并且我发送消息确认你是Primary。接下来我开始回答其他询问Primary的客户，并说那边哪个是Primary，而与此同时给您的消息仍在传输中，当你收到消息时，消息标识的Primary已经是旧的了。如果没有更聪明的机制，你不可能意识到即使你从Master那里得到了这些信息，但它已经过时了。如果此时你（前一个Client）发送修改数据的请求，并且成功返回了，那么我们有了两个冲突的副本。

- Q8: you've a new file and no replicas, so if you have a new file no replicas or even an existing file and no replicas the you'll take the path code. ↓

如果你有一个新文件，没有副本，或者现有文件没有副本，你将使用之前提到的路径代码。↓

<img width="1291" alt="image" src="https://user-images.githubusercontent.com/24760084/180413754-f44b3220-02b1-40cd-a4dd-4c4435e8b9b6.png">

<img width="907" alt="image" src="https://user-images.githubusercontent.com/24760084/180413801-7fe650ce-3b1f-448c-accb-ea0fb60f2eee.png">

the master will receive a request from a client saying, I'd like to append to this file and then the master will first see there's no chunks associated with that file, and it will make up a new chunk identifier or perhaps by calling the random number generator and then it'll look in its chunk information table and say gosh I don't have any information about that chunk and it'll make up a new record saying but it must be special case code where it says well I don't know any version number this chunk doesn't exist, I'm just gonna make up a new version number, one pick a random primary and set of secondaries and tell them look you are responsible for this new empty chunk, please get to work. the paper says three replicas per chunk by default, so typically a primary and two backups.

Master 将收到来自客户端的请求说，我想附加到这个文件，然后 master 将首先发现没有与该文件关联的Chunk，于是Master将组成一个新的块标识符，或者可能通过调用随机数字生成器，然后它会查看它的块信息表（chunk table）并说天哪，我没有关于那个块的任何信息，它会组成一条新记录，并它必须是特殊情况代码来支持，Master同样不知道这个块的任何版本号，再编写一个新的版本号，继续随机选择一个Primary和一组Secondaries，最后告诉它们来负责这个新的空Chunk，请开始工作。该论文说默认情况下每个块有三个副本，因此通常是一个主副本和两个备份。

---这实际就等同于一次，GFS对添加的新文件的初始化过程。

下一节，对GFS 一致性的分享。



