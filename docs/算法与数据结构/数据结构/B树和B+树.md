# B树和B+树

## 哈希索引

基于哈希表实现的哈希索引

在Mysql的InnoDB引擎中，自适应哈希索引就是用哈希表实现的。

哈希索引是数据库自身创建并使用的，DBA本身不能对其进行干预，但是可以通过参数来禁止或者启用此特性。

显然用哈希表实现索引的好处是非常明显的，查找单个指定数据只需要O(1)的时间复杂度。

但是因为哈希表本身是无序的，所以不利于范围查询。

## 二叉查找树
二叉查找树（Binary Search Tree）即BST树是这样的一种数据结构,如下图

![](/uploads/upload_a8ec5629cb022611c316a1f2763b20f7.png)


在二叉搜索树中：

1).若任意结点的左子树不空，则左子树上所有结点的值均不大于它的根结点的值。

2).若任意结点的右子树不空，则右子树上所有结点的值均不小于它的根结点的值。

3).任意结点的左、右子树也分别为二叉搜索树。

这样的结构非常适合用二分查找的思维查找元素。

但是当二叉树的构造变成这样时,

![](/uploads/upload_6621aadd300a9d26580c165b3d29a36b.png)


此时我们再查找8时,查找效率就沦为接近顺序遍历查找的效率。

## 平衡二叉树 AVL

**我们对二叉查找树做个限制，限制必须满足任何节点的两个子树的最大差为1**，也是平衡二叉树的定义，这样我们的查找效率就有了一定的保障。

当数据量很多时，同样也会出现二叉树过高的情况。

我们知道平衡二叉树的查找效率为 O(log n)，也就是说，当树过高时，查找效率会下降。

另外由于我们的索引文件并不小，所以是存储在磁盘上的。

文件系统需要从磁盘读取数据时，一般以页为单位进行读取，假设一个页内的数据过少， 那么操作系统就需要读取更多的页，涉及磁盘随机I/O访问的次数就更多。

将数据从磁盘读入内存涉及随机I/O的访问，是数据库里面成本最高的操作之一。

因而这种树高会随数据量增多急剧增加，每次更新数据又需要通过左旋和右旋维护平衡的二叉树，不太适合用于存储在磁盘上的索引文件。

## 更加符合磁盘特征的B树

前面我们看到，虽然平衡二叉树既有链表的快速插入与删除操作的特点，又有数组快速查找的优势，但是这并不是最符合磁盘读写特征的数据结构。

也就是说，我们要找到这样一种数据结构，能够有效的控制树高，那么我们把二叉树变成m叉树，也就是下图的这种数据结构:B树。

B树是一种这样的数据结构：

![](/uploads/upload_753b7ebf5d8bfb1da3367e4a7ffcbebe.png)

1.根结点至少有两个子结点;

2.每个中间节点都包含k-1个元素和k个子结点，其中 m/2 <= k <= m;

3.每一个叶子结点都包含k-1个元素，其中 m/2 <= k <= m;

4.所有的叶子结点都位于同一层;

5.每个结点中关键字从小到大排列，并且当该结点的孩子是非叶子结点时，该k-1个元素正好是k个子结点包含的元素的值域的分划。

可以看到，B树在保留二叉树预划分范围从而提升查询效率的思想的前提下，做了以下优化：

二叉树变成m叉树，这个m的大小可以根据单个页的大小做对应调整，从而使得一个页可以存储更多的数据，从磁盘中读取一个页可以读到的数据就更多，随机IO次数变少，大大提升效率。

**但是我们看到，我们只能通过中序遍历查询全表，当进行范围查询时，可能会需要中序回溯**

## B+树

![](/uploads/upload_eb8666e3f58c85d76b371580ca79f748.png)

B+树在B树的基础上加了以下优化：

1.叶子结点增加了指针进行连接，即叶子结点间形成了链表；

2.非叶子结点只存关键字key，不再存储数据，只在叶子结点存储数据；

说明：叶子之间用双向链表连接比单向链表连接多出的好处是通过链表中任一结点都可以通过往前或者往后遍历找到链表中指定的其他结点。

这样做的好处是：

1.范围查询时可以通过访问叶子节点的链表进行有序遍历，而不再需要中序回溯访问结点。

2.非叶子结点只存储关键字key，一方面这种结构相当于划分出了更多的范围，加快了查询速度，另一方面相当于单个索引值大小变小，同一个页可以存储更多的关键字，读取单个页就可以得到更多的关键字，可检索的范围变大了，相对IO读写次数就降低了。

**B+ 树和 B 树的区别？**

1. B树非叶子结点和叶子结点都存储数据,因此查询数据时，时间复杂度最好为O(1),最坏为O(log n)。

    B+树只在叶子结点存储数据，非叶子结点存储关键字，且不同非叶子结点的关键字可能重复，因此查询数据时，时间复杂度固定为O(log n)。

2. B+树叶子结点之间用链表相互连接，因而只需扫描叶子结点的链表就可以完成一次遍历操作，B树只能通过中序遍历。

**为什么 B+ 树比 B 树更适合应用于数据库索引？**

1. B+树更加适应磁盘的特性，相比B树减少了I/O读写的次数。由于索引文件很大因此索引文件存储在磁盘上，B+树的非叶子结点只存关键字不存数据，因而单个页可以存储更多的关键字，即一次性读入内存的需要查找的关键字也就越多，磁盘的随机I/O读取次数相对就减少了。

2. B+树的查询效率相比B树更加稳定，由于数据只存在在叶子结点上，所以查找效率固定为O(log n)。

3. B+树叶子结点之间用链表有序连接，所以扫描全部数据只需扫描一遍叶子结点，利于扫库和范围查询；B树由于非叶子结点也存数据，所以只能通过中序遍历按序来扫。也就是说，对于范围查询和有序遍历而言，B+树的效率更高。