适用于查找和修改

删除最后一个元素不一定要真的删除，只要让size变小，而当有人试图访问size以外的位置可以让他抛出异常。洞穴预言
![[Screenshot_20250901_093933_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]

但是，如果是一个泛型数组，则要将其设置为null，告诉Java的垃圾回收机制这个可以被回收了

怎么解决数组大小的问题？创建一个比原数组大的数组，将数据复制，但是这里的复制操作会极大加大内存的使用，并且耗时。
![[Screenshot_20250901_095240_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]


这个耗时差异主要是由于数组链表每次都要复制数组导致的（100+101+102+...+100000=(100+100000)*99900/2）

![[链表结构和数组结构在加入大量数据的情况下的耗时差异.jpg]]

解决方法：
方式一：size加某个数，只是让抛物线形的增长规律变得更后面出现
方式二：size乘以某个因子，可以有效解决这个问题

![[Screenshot_20250901_101247_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]


而删掉的进行，当数组只剩下四分之一的时候换到一个小数组
为什么不是二分之一？因为这样你一加一个数据又要进行一个扩大。