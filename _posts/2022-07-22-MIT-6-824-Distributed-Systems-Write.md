![wirte](https://pica.zhimg.com/v2-15016cef42422ccb60b8a842efe41fee_1440w.jpg?source=172ae18b)
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

一种情况是 没有主副本。 在这种情况下，master 需要找出具有该chunk 的最新数据副本的一组块服务器(包含primary和secondary replica)，因为如果您已经运行系统很长时间，由于故障或任何可能的原因，那里有块的旧副本的块服务器，比如从昨天或上周开始断更的，尽管我一直在更新数据。因为可能该服务器已经宕机了几天并且没有接收更新，所以你需要能够区分块的最新副本和非最新副本之间的区别。
