
# 布谷鸟过滤器（Cuckoo Filter）

## 前提

在我们工作中，如果需要判断一个数据是否存在于一个集合中，通常需要将所有数据保存起来（数据库，各类缓存），再通过查询和比较确定。

此时如果请求量过大，则数据库容易出现性能瓶颈，甚至被打挂；如果追求性能而采用缓存，需要的存储空间也会随元素量呈线性增长，同样可能到达瓶颈。

布隆过滤器则是用于解决这一需求的方案之一。其包含一个固定大小的二进制向量或者位图（bitmap）。插入请求时，将元素的签名（fingerprint）进行多次哈希，再映射到该位图中的多个点位。

查询请求时，则再次进行相同的哈希运算，若被查询元素对应的点位均存在，则元素可能存在；若至少有一个对应点位不存在，则元素必然不存在。

该算法的插入/查询时间都是常数 O(K)，且大大减少了空间成本。但布隆过滤器同样存在几个主要缺陷：
1. 不支持删除元素：一旦对位数组进行了赋值，无法将其删除（即使位图的点位不使用bit，而采取计数器，也存在误判导致误删的问题）。
2. 查询性能弱：布隆过滤器使用多个hash函数计算位图多个不同位点，由于多个位点在内存中不连续，CPU寻址花销较大。
3. 空间利用率低。

## 布谷鸟过滤器

在论文《Cuckoo Filter: Practically Better Than Bloom》中，直接提及了该过滤器的四大优点：
>1. It supports adding and removing items dynamically;
>2. It provides higher lookup performance than traditional Bloom filters, even when close to full (e.g., 95% space utilized);
>3. It is easier to implement than alternatives such as the quotient filter; and
>4. It uses less space than Bloom filters in many practical applications, if the target false positive rate is less than 3%.

翻译如下：
1. 支持动态的新增和删除元素；
2. 提供了比传统布隆过滤器更高的查找性能，即使在接近满的情况下（比如空间利用率达到 95% 的时候）；
3. 比起某些替代方案，如商过滤器（quotient filter），更容易实现；
4. 如果要求错误率小于3%，在许多实际应用中，它比布隆过滤器占用的空间更小。

### 命名来源

布谷鸟过滤器源于布谷鸟哈希算法，布谷鸟哈希算法源于生活 —— 那个热爱「鸠占鹊巢」的布谷鸟。布谷鸟喜欢滥交（自由），从来不自己筑巢。它将自己的蛋产在别人的巢里，让别人来帮忙孵化。待小布谷鸟破壳而出之后，因为布谷鸟的体型相对较大，它又将养母的其它孩子（还是蛋）从巢里挤走 —— 从高空摔下夭折了。

### 布谷鸟哈希

建立两个 hash 表T1、T2，两个 hash 表对应两个 hash 函数 H1、H2。

插入步骤如下：

1. 当一个不存在的元素插入的时候，会先根据 H1 计算出其在 T1 表的位置，如果该位置为空则可以放进去。
2. 如果该位置不为空，则根据 H2 计算出其在 T2 表的位置，如果该位置为空则可以放进去。
3. 如果T1 表和 T2 表的位置元素都不为空，那么就随机的选择一个 hash 表将其元素踢出。
4. 被踢出的元素会循环的去找自己的另一个位置，如果被暂了也会随机选择一个将其踢出，被踢出的元素又会循环找位置；
5. 如果出现循环踢出导致放不进元素的情况，那么会设置一个阈值，超出了某个阈值，就认为这个 hash 表已经几乎满了，这时候就需要对它进行扩容，重新放置所有元素。

举一个例子来说明：
![](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210228115756753-1960084033.png)

如果想要插入一个元素Z到过滤器里：

1. 首先会将Z进行 hash 计算，发现 T1 和 T2 对应的槽位1和槽位2都已经被占了；
2. 随机将 T1 中的槽位1中的元素 X 踢出，X 的 T2 对应的槽位4已经被元素 3 占了；
3. 将 T2 中的槽位4中的元素 3 踢出，元素 3 在 hash 计算之后发现 T1 的槽位6是空的，那么将元素3放入到 T1 的槽位6中。

当 Z 插入完毕之后如下：
![](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210228115757450-332206295.png)

### 优化

上面的布谷鸟哈希算法的平均空间利用率并不高，大概只有 50%。到了这个百分比，就会很快出现连续挤兑次数超出阈值。这样的哈希算法价值并不明显，所以需要对它进行改良。

改良的方案之一是增加 hash 函数，让每个元素不止有两个巢，而是三个巢、四个巢。这样可以大大降低碰撞的概率，将空间利用率提高到 95%左右。

另一个改良方案是在数组的每个位置上挂上多个座位，这样即使两个元素被 hash 在了同一个位置，也不必立即「鸠占鹊巢」，因为这里有多个座位，你可以随意坐一个。除非这多个座位都被占了，才需要踢出元素。很明显这也会显著降低碰撞次数。这种方案的空间利用率只有 85%左右，但是查询效率会很高，同一个位置上的多个座位在内存空间上是连续的，可以有效利用 CPU 高速缓存。

更加高效的方案是将上面的两个改良方案融合起来，比如使用 4 个 hash 函数，每个位置上放 2 个座位。这样既可以得到时间效率，又可以得到空间效率。这样的组合甚至可以将空间利用率提到高 99%，这是非常了不起的空间效率。

## 布谷鸟过滤器

布谷鸟过滤器也是一维数组，但是不同于布谷鸟哈希的是，布谷鸟哈希会存储整个元素，而布谷鸟过滤器中只会存储元素的指纹信息（fingerprint，几个bit，类似于布隆过滤器）。这里过滤器牺牲了数据的精确性换取了空间效率。正是因为存储的是元素的指纹信息，所以会存在误判率，这点和布隆过滤器如出一辙。

首先布谷鸟过滤器还是只会选用两个 hash 函数，但是每个位置可以放置多个座位。问题在于，过滤器中只存储指纹信息，但而计算元素的另一个位置需要元素本身，例如：
```markdown
byte fp = fingerprint(x)
int pos1 = hashCode1(x) % length
int pos2 = hashCode2(x) % length
```
只知道了 p1 和 x 的指纹，是没办法直接计算出 p2 的。

#### 解法
布谷鸟过滤器巧妙的地方就在于设计了一个独特的 hash 函数，使得可以根据 p1 和 元素指纹 直接计算出 p2，而不需要完整的 x 元素。
```markdown
byte fp = fingerprint(x)
int pos1 = hashCode(x)
int pos2 = pos1 ^ hashCode(fp)  // 异或运算

根据XOR的性质可得：
int pos1 = pos2 ^ hashCode(fp)
```
所以我们根本不需要知道当前的位置是 pos1 还是 pos2，只需要将当前的位置和 hashCode(fp) 进行异或计算就可以得到对偶位置。而且只需要确保 hashCode(fp) != 0 就可以确保 pos1 != pos2，如此就不会出现自己踢自己导致死循环的问题。

也许你会问为什么这里的 hashCode 函数不需要对数组的长度取模呢？实际上是需要的，但是布谷鸟过滤器强制数组的长度必须是 2 的指数，所以对数组的长度取模等价于取 hash 值的最后 n 位。在进行异或运算时，忽略掉低 n 位 之外的其它位就行。将计算出来的位置 pos 保留低 n 位就是最终的对偶位置。

### 数据结构

简单起见，我们假定指纹占用一个字节，每个位置有 4 个 座位。

```
public class cuckoo_filter {
	private static final int DEFAULT_SEATS = 4;
	private byte[][] map = new byte[DEFAULT_LENGTH][DEFAULT_SEATS] ; 	// 每个位置包含4个座位
	private int elements;		// 容纳的元素的个数
	private int maxKick; 		// 最大挤兑次数
}
```

### 插入算法

插入需要考虑到最坏的情况，那就是挤兑循环。所以需要设置一个最大的挤兑上限

```
public boolean insert(x) {
   byte fp = fingerprint(x);
   int pos1 = hashCode(x);
   int pos2 = pos1 ^ hashCode(fp);
  // 尝试加入任意一个位置的任意空坐席
  if (inserted(map[pos1], fp) || inserted(map[pos2], fp)) {
     nums++;
     return true;
  }
  // 随机选择一个位置
  int kickPos = Math.random() > 0.5 ? pos1 : pos2;
  int kickCount = 0;
  while (kickCount < maxKick){
	// 踢出任意元素
	byte old_fp = replaced(map[pos], fp);
	fp = old_fp;
	// 计算另一个位置
    pos = pos ^ hashCode(fp)
	// 尝试加入另一个位置的空坐席
    if (inserted(map[pos], fp)){
	  nums++;
      return true;
	}
	// 未成功，继续踢出元素
    kickCount++;
  }
  return false

```
![](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210228115758266-1062582962.png)

### 查找算法

查找非常简单，在两个 hash 位置的桶里找一找有没有自己的指纹就 ok 了。

```
public boolean contains(x) {
   byte fp = fingerprint(x);
   int pos1 = hashCode(x);
   int pos2 = pos1 ^ hashCode(fp);
   return contains(map[pos1], fp) || contains(map[pos2], fp);
}
```

### 删除算法

删除算法和查找算法差不多，也很简单，在两个桶里把自己的指纹抹去就 ok 了。

```
public boolean delete(x) {
   byte fp = fingerprint(x);
   int pos1 = hashCode(x);
   int pos2 = pos1 ^ hashCode(fp);
   boolean success = deleted(map[pos1], fp) || deleted(map[pos2], fp);
   if (success) nums--;
   return ok
}
```
  
###   缺点

So far so good！布谷鸟过滤器看起来很完美啊！删除功能和获取元素个数的功能都具备，比布隆过滤器强大多了，而且似乎逻辑也非常简单，上面寥寥数行代码就完事了。如果插入操作返回了 false，那就意味着需要扩容了，这也非常显而易见。

然而不妨想一下，如果布谷鸟过滤器对同一个元素进行多次连续的插入会怎样？

毫无疑问，这个元素的指纹会霸占两个位置上的所有座位 —— 8个座位。这 8 个座位上的值都是一样的，都是这个元素的指纹。如果继续插入，则会立即出现挤兑循环。从 p1 槽挤向 p2 槽，又从 p2 槽挤向 p1 槽。

也许你会想到，能不能在插入之前做一次检查，询问一下过滤器中是否已经存在这个元素了？这样确实可以解决挤兑循环的问题，但是删除的时候会出现高概率的误删。

简单来说，如果两个元素被 hash 到同一个位置，且两个元素的指纹相同，那么插入检查会认为它们是相等的，此时删除任意一个元素，都将导致对另一个元素的查询返回错误的“不存在”判断（false negative)，这显然是难以接受的。

论文中也提到了这个问题，见下图： 

![](https://mmbiz.qpic.cn/mmbiz_png/bGribGtYC3mIh7yriajtOedrOJBTc6ja34uMVzGbicXYzWq6yfBaCUX4Lbicib16oeiaeOEnZibuHficg76EDYiaRJicicgnw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这句话明确告诉我们如果想要让布谷鸟过滤器支持删除操作，那么就必须不能允许插入操作多次插入同一个元素，确保每一个元素不会被插入多次（kb+1）。这里的 k 是指 hash 函数的个数 2，b 是指单个位置上的座位数，这里我们是 4。

在现实世界的应用中，确保一个元素不被插入指定的次数那几乎是不可能做到的。因为要想做到，则必须维护一个外部的字典来记录每个元素的插入次数呢——那这个外部字典的存储空间怎么办？

而因为不能完美的支持删除操作，也就无法较为准确地估计内部的元素数量。

来自CuckooFilter4j 的特别说明
大意：布谷鸟过滤器可以在不消耗额外空间的条件下，且保证相同元素数量有限（<8 ~ 9)的情况下，达成删除元素。但删除不存在的元素可以导致 false negative。
![](https://mmbiz.qpic.cn/mmbiz_jpg/bGribGtYC3mIh7yriajtOedrOJBTc6ja34e3CDT1myiagWgjIYNBFcTnkGbgibEtuqUYGAF0OJvAgbPcuNhmoGvnFA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
