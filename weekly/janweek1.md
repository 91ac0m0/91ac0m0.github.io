### JAN week 1

#### bins 的理解

一直觉得堆的各种机制很混乱，就对着[这篇文章](https://0x434b.dev/overview-of-glibc-heap-exploitation-techniques/)的的 basic 部分才从一个比较全局的视角去理解堆的分配机制的框架是怎样的。

一个重要的点是理解 bins 概念。free 了chunk 之后，bins 就像垃圾分类一样把 chunk 组织到对应的箱子里。

分类的依据首先是大小：fastbins、smallbins、largebins 的大小依次增大，碎片大小的 free chunk 进入 fasbins 便于快速分发，再大一点的 chunk 进入 smallbins，更大一点进入 largebins。

bins 的分配也要遵循一定顺序，chunk free 之后较小的 chunk(0x20-0xb0)进入 fastbins，不在这个范围内就进入 unsorted，后续再整理时 unsotedbins 中的小的进 smallbins，大的进 largebins。较高版本 glibc ($\geq 2.26 $)的一个重要改变是引入 tcache，满足 tcache 大小($\leq 0x410$)的chunk 首先进入 tcache，每个链的上限是 7 个，如果同一个大小的 chunk free 超过 7 个，就会进入其它 bins。

malloc chunk 时在堆中的搜索顺序是 tcache, fastbins and smallbins, unsortedbins largebins。

最后，各个 bins 的数据结构因各自功能不同而各异，不同版本之间规则、检查也不一样，还是有点点乱，得动手调才能明白的感觉。

#### largebin attack

单纯的 largebin attack 就是向任意地址写入所在的堆地址，需要配合其他操作才能实现任意写。参考[这篇文章](https://xz.aliyun.com/t/5177)看了下源码，然后就是调 how2heap。

