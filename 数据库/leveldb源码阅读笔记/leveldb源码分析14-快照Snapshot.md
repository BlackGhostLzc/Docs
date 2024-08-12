## **Snapshot**

snapshot，即快照，赋予了使用者查找在过去某一个瞬间数据库状态的能力：在那一时刻，整个 leveldb 有哪些数据，什么版本。 leveldb 会冗余地存储很多旧版本的数据。我们有可能误删了数据， 或者想追踪数据的变化历史，这时快照功能就十分有用了。 实现快照功能有以下两个主要问题：

1. 前面说过，应该冗余存储旧版本数据。那么应该保留多老的数据呢？ 联想到 leveldb 中任何一个数据(键值对)都有一个整数版本号 sequence number，我们也可以用一个整数来告诉 leveldb 对每个 user_key 都应该**至少保留一个小于这个整数的版本**(如果这个 user_key 有小于此整数的版本的话，大于这个版本的也都需要进行存储)。这个整数即是 snapshot。 只要 user_key 在那个时刻(产生 snapshot 这个版本的时刻)有版本(即 user_key 存在小于等于 snapshot 的版本)， 则我们应该返回这个时刻当 时最新的版本，即 user_key 的所有版本中小于等于 snapshot 的最大版本。 snapshot 的值必然比当前 leveldb 最新版本号小：最新版本号时时刻刻都是整个 leveldb 中最大的版本号。
2. 我们应该如何使用 snapshot 以对每个 user_key 保存一个小于等于它的版本呢？我们在每次新写入时不可能做到这点，最新版本号作为 snapshot 没有意义：违背了快照的初衷，快照是历史的某时刻状态，并非现在。除了新写入数据，后面唯一一个可以动数据的地方就是 compaction 了。 leveldb 通过在 compaction 时给出一个 snapshot，在执行 compaction 时，会为每个 user_key 保留一个小于等于 snapshot 的版本(如果有的话)。

那么如何使用快照呢？ -- 查询数据时，通过option传入snapshot参数，查找时会跳过版本号比 snapshot 大的键值对，定位到小于等于 snapshot 的最新版本， 从而读取那个时刻的历史数据。





## **Compact && Snapshot**

前面我们讲过Compaction部分的代码，选择出待 compaction 的文件：level 层和 level+1 层，将这两个层次的 sst 文件包装为一个 MergingIterator，按序遍历键值对，一个一个判断，是 drop 掉，还是加入到新的 sst 文件中。

这里也是对上面Compaction进行drop部分的一个小的补充。

1. 对于当前遍历到的 user_key，为它保留了一个小于等于指定 snapshot 的最新版本。
2. 首次遍历到的 user_key，不论版本号与 snapshot 关系如何，都应该保留下来，即使当前版本是一个 deletion，也应保留。

> a).这个 user_key 最新版本的值类型是 deleteType：则这个 user_key 在之后的遍历中都不应该再被查找到。但是如果删除了这个 deletion 版本，此 user_key 的旧版本会被遍历，而一旦这个旧版本是一个正常的 valueType，则它会被查找到返回给用户，这是逻辑错误。因此如果要删除这个 deletion 版本，则需要删除此 user_key 的所有版本：让这个 user_key 不可能被查到。基于这点，有以下两个分支判断： 
>
> a-1).这个 deletion 版本的版本号小于等于 snapshot ：删除所有的历史版本，相当于人为地扼杀了 user_key 本应该具有的快照功能。 
>
> a-2).这个 deletion 版本的版本号大于 snapshot，有以下两个分支判断： 
>
> a-b-1)若后续的遍历中 user_key 存在小于等于 snapshot 的版本，则与上面 a-1) 一样，扼杀了快照功能。
>
> a- b-2)若后续的遍历中 user_key 不存在小于等于 snapshot 的版本，并没有扼杀对于 snapshot 的快照功能。弊端不很明显，在于，compaction 指定的 是一个版本号数组，虽然没有扼杀最小的 snapshot 的快照功能，但是有可能扼杀较大的 snapshot 的快照功能。不扼杀的情况是：此 user_key 的最老 的版本号大于最大的 snapshot。但是对于其它的 user_key，可能仍然会扼杀快照功能。完全搞清楚：是否对于所有的 user_key 没有扼杀任何一个 snapshot 的快照功能，是一件非常繁杂的事情。因此简单而且稳妥起见，不要 drop 这个 deletion 版本即可。

3. 非首次遍历到的 user_key 是一个 deletion 版本，则 drop 依赖于以下两个条件满足： 
   - 当前 user_key 只出现在执行 compaction 的层次(设为 level) 和 level+1 层中 
   - 当前遍历的版本号小于等于 snapshot

> 1.当前是要将 level_ 层和 level_+1 层 compact，虽然在这两层将 user_key 的这个 deletion 版本删除了，但如果当前 user_key 还出现在比 level+1 层更高的层中，这些稍老的版本（层次越高数据越老越“冷”）会响应对 user_key 的查找，这将是逻辑错误。 2.若当前这个 deletion 版本号大于 snapshot 仍然将它 drop 掉，为了保持逻辑的正确性，还需要将此 user_key 后续版本全部 drop（第2点中讲过）。但 这样也有问题：会扼杀快照功能（上面详细阐述了这点）