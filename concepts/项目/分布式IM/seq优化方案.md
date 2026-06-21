		Seq优化方案

# 1、seq优化方案时序图


# 2、关于会话的seq处理流程

1~3、消息触发从redis申请seq，使用INCR命令
4、判断计数器值seq
5~9、判断redis获取到的不同seq值分情况处理：
if(seq%10000==0){//到达预分配极限}
else if(seq == 1){//这时候可能是会话刚开始第一次申请，也可能redis变化过}
else{//redis返回的计数器值直接可用}

# 3、 优化方案说明

本次seq优化方案，不再依赖业务集群结点内存。利用redis本身的高可用，直接使用redis计数器功能，配合mongo持久化预分配号段以应对redis重启。Seq最终效果为“从1开始不断增长”

# 4、存在的问题（上限/升级）

	1、seq无限增长，理论会到达int64极限
	2、seq无限增长，对于seq这个功能本身的升级带来一定困难，需要考虑当前系统的seq是否支持升级	(缓存清空）
3、带版本号的seq可能自由度更高
4、当前系统seq来源的第一步都是读redis，redis的数据清空目前都会触发一次预分配，所以当前系统可以升级改造后的seq方案
5、seq改造后，群聊可以直接升级，单聊seq无法很好按讨论对双方采用各自seq，因为持久化的单聊seq目前不是以userId来的，是按会话来的。除了为了升级在代码里做特殊处理（两人公用同一份seq,创造出持久化里另一userId的seq），等升级后要再修改为正常逻辑。

# 5、seq基于单一存储集群实现（方案1）

a.基于codis的可靠性， 采用INCR命令申请seq；基于redis的持久化, 当redis重启也可往内存恢复相应的seq值，然后接着重启前的seq值再继续往上增长。纯粹使用codis就可以保证seq的可用性
b.利用mongodb的可靠性，采用独立的一张表保存seq, 使用$inc命令往mongo申请seq.
只要mongo不出现灾难性数据丢失，seq则一直可用
注：首先利用mongo的原子计数指令{$inc}获取seq, 然后使用申请到seq写库。
c.以上两种方案都是基于单独的存储集群获得seq，只是seq不可回退的限制导致了当灾难性数据丢失时，也就无法保证系统的高可用性

# 6、seq基于codis和mongo存储集群实现（方案2）

seq 使用64位长整型
分配seq：
- 2向codis集群请求seq：请求分配会话的seq（使用lua原子操作）。
- 1）没有key，返回{-1,0}；
- 2）有key，判断seq的值（seq%10000 == 0），但是需要更新mongo的seq，返回{-2,0}
- 3）有key，（seq%10000 != 0），incr 1,返回后面的值{0,curseq}；（使用该值作为seq）
- 情况1）（codis没有key）
- 4 向mongo集群分配seq：请求分配会话的seq，会话id对应的seq增加10000并返回（incrby 10000）之前的值（没有则认为0）。mongo没有则插入。（如果并发请求导致mongo seq多往后移动，不影响使用）
- 7 回写seq到codis：把seq回写到codis集群，未存在key的则设置，已存在的key则不设置（setnx），然后递增incr并返回后面的值{curseq}
- 情况2）（codis有key，但是要更新mongo）
- 10更新seq的mongo分配起点:更新mongo的seq = curseq + 10000 的更新请求到mongo。mongo没有则插入（本情况一般不存在，如mongo被清除，但是codis有key）。
- 12 再次向codis集群请求seq：请求分配会话的seq（使用lua原子操作）。
- 没有key，返回{-1,0}；（分配失败，本请求不再继续申请seq）
- 2）有key，incr 1,返回后面的值{0,curseq}；（使用该值作为seq）
Mongo结构
会话消息序号 session_msgseq
作用：保存会话的消息序号下一个分配起点

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| sessionId | string | 8 | 会话id |
| msgseqStart | Int64 |  | 消息序号起点 |
| updatetime | Int64 |  | 更新时间（微妙） |

Redis结构
会话用户消息序号(string)
4:1:im:msgseq:sessionid:1214152769825210369