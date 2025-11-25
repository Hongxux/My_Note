LFU算法实现的关键在于如何能高效的保存和读取数据最近的访问频次信息，通常的做法是使用popularity sketch(一种概率数据结构)来识别数据的"命中"事件，从而记录数据的访问次数。CountMin-Sketch便是其中的一种，他是通过一个计数矩阵和多个哈希算法实现的，如图所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92be2d401d434df2867e45f26897827e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

CountMin Sketch的原理类似于[[布隆过滤]]，也是一种概率型的数据结构。其中不同的row对应着不同的哈希算法，depth大小代表着哈希算法的数量，width则表示数据可哈希的范围。当记录某个指定key的访问次数时，分别使用不同的哈希算法在其对应的row上做哈希操作，如果命中了某一个数据格，则将该数据格的引用计数+1。**当查询某个指定key的访问次数时，经过哈希定位到具体的多个数据格后，返回最小的数量计为该数据的访问次数**。使用多个哈希算法可以降低哈希碰撞带来的数据不准确的概率，宽度上的增加可以提高key的哈希范围，减少碰撞的概率，因此我们可以通过调整矩阵的width和depth达到算法在空间、效率和哈希碰撞产生的错误率之间平衡的目的。Caffeine中的CountMin Sketch是通过四种哈希算法和一个long型数组实现的，具体的实现方法可以参考咖啡拿铁大神的文章——[深入解密来自未来的缓存-Caffeine](https://juejin.im/post/5b8df63c6fb9a019e04ebaf4 "https://juejin.im/post/5b8df63c6fb9a019e04ebaf4")，

首先要说到的就是频率记录的问题，我们要实现的目标是利用有限的空间可以记录随时间变化的访问频率。在W-TinyLFU中使用Count-Min Sketch记录我们的访问频率，而这个也是布隆过滤器的一种变种。如下图所示:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27e192f86bda4b9c86f00bddca52771a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

如果需要记录一个值，那我们需要通过多种Hash算法对其进行处理hash，然后在对应的hash算法的记录中+1，为什么需要多种hash算法呢？由于这是一个压缩算法必定会出现冲突，比如我们建立一个Long的数组，通过计算出每个数据的hash的位置。比如张三和李四，他们两有可能hash值都是相同，比如都是1那Long[1]这个位置就会增加相应的频率，张三访问1万次，李四访问1次那Long[1]这个位置就是1万零1，如果取李四的访问评率的时候就会取出是1万零1，但是李四命名只访问了1次啊，为了解决这个问题，所以用了多个hash算法可以理解为long[][]二维数组的一个概念，比如在第一个算法张三和李四冲突了，但是在第二个，第三个中很大的概率不冲突，比如一个算法大概有1%的概率冲突，那四个算法一起冲突的概率是1%的四次方。通过这个模式我们取李四的访问率的时候取所有算法中，李四访问最低频率的次数。所以他的名字叫Count-Min Sketch。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/986a47766e7f475da1e918e5df67828a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12d33b3e43b24513b1ddde9829ee1e2b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里和以前的做个对比，简单的举个例子:如果一个hashMap来记录这个频率，如果我有100个数据，那这个HashMap就得存储100个这个数据的访问频率。哪怕我这个缓存的容量是1，因为Lfu的规则我必须全部记录这个100个数据的访问频率。如果有更多的数据我就有记录更多的。

**在Count-Min Sketch中，我这里直接说caffeine中的实现吧(在FrequencySketch这个类中),如果你的缓存大小是100，他会生成一个long数组大小是和100最接近的2的幂的数，也就是128**。而这个数组将会记录我们的访问频率。在caffeine中规定频率最大为15，15的二进制位1111，总共是4位，而Long型是64位。所以每个Long型可以放16种算法，**但是caffeine并没有这么做，只用了四种hash算法，每个Long型被分为四段，每段里面保存的是四个算法的频率**，这样做的好处是可以进一步减少Hash冲突，原先128大小的hash，就变成了128X4。 一个Long的结构如下:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a42b0132bfef45d6b81dcce8d36e3ff3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

我们的4个段分为A,B,C,D，在后面我也会这么叫它们。而每个段里面的四个算法我叫他s1,s2,s3,s4。下面举个例子如果要添加一个访问50的数字频率应该怎么做？我们这里用size=100来举例。

1. 首先确定50这个hash是在哪个段里面，通过hash & 3(3的二进制是11)必定能获得小于4的数字，假设hash & 3=0，那就在A段。
2. 对50的hash再用其他hash算法再做一次hash，得到long数组的位置，也就是在长度128数组中的位置。假设用s1算法得到1，s2算法得到3，s3算法得到4，s4算法得到0。
3. 因为S1算法得到的是1，所以在long[1]的A段里面的s1位置进行+1,简称1As1加1，然后在3As2加1，在4As3加1，在0As4加1。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d49e2960ef247ed976ef7f4e646310d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这个时候有人会质疑频率最大为15的这个是否太小？没关系在这个算法中，比如size等于100，如果他全局提升了size*10也就是1000次就会全局除以2衰减，衰减之后也可以继续增加，这个算法再W-TinyLFU的论文中证明了其可以较好的适应时间段的访问频率。