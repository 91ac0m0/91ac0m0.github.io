### FEB week 1

（happy 了好几天，没看习题安抚

注释了一下 glibc 2.32 的 malloc.c 



#### 几个重要的结构体

##### malloc_chunk

最基础的结构（除了 tcache chunk），是其他结构体的成员

```bash
pwndbg> ptype /o mchunkptr
type = struct malloc_chunk {
/*    0      |     8 */    size_t mchunk_prev_size;
/*    8      |     8 */    size_t mchunk_size;
/*   16      |     8 */    struct malloc_chunk *fd;
/*   24      |     8 */    struct malloc_chunk *bk;
/*   32      |     8 */    struct malloc_chunk *fd_nextsize;
/*   40      |     8 */    struct malloc_chunk *bk_nextsize;

                           /* total size (bytes):   48 */
                         } *
```



##### malloc_state

```bash
pwndbg> ptype /o mstate
type = struct malloc_state {
/*    0      |     4 */    __libc_lock_t mutex;
/*    4      |     4 */    int flags;
/*    8      |     4 */    int have_fastchunks;
/* XXX  4-byte hole  */
/*   16      |    80 */    mfastbinptr fastbinsY[10];
/*   96      |     8 */    mchunkptr top;
/*  104      |     8 */    mchunkptr last_remainder;
/*  112      |  2032 */    mchunkptr bins[254];
/* 2144      |    16 */    unsigned int binmap[4];
/* 2160      |     8 */    struct malloc_state *next;
/* 2168      |     8 */    struct malloc_state *next_free;
/* 2176      |     8 */    size_t attached_threads;
/* 2184      |     8 */    size_t system_mem;
/* 2192      |     8 */    size_t max_system_mem;

                           /* total size (bytes): 2200 */
                         } *
```

malloc_state 是 ptmalloc 中的所有的堆的管理结构。其中的 fastbinsY[10]数组 储存 fastbins 的链表头，bins[254] 数组储存 unsorted、small、large bins 的链表头。bins[256]，由 128 对 fd 和 bk 组成。

如图：

```txt
            +-----------+-----------+
bin_at(i)-->| fd(i-1)	|  bk(i-1)	|
            +-----------+-----------+
            |   fd(i)   |   bk(i)   |
            +-----------+-----------+
            |  fd(i+1)  |  bk(i+1)  |
            +-----------+-----------+
```

分别是 1 对 unsortedbins 的 fd 和 bk，62 对 smallbins 的，64 对 largebins。 

![](https://img-blog.csdnimg.cn/20201111105215902.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2luaXRwaHA=,size_16,color_FFFFFF,t_70)

当要寻找某一 bins 时，就通过 bin_at 宏取出 bins[] 数组（注意下标 i 从 1 开始），返回的结果转换成 malloc_chunk (mbinptr)，刚好使 fd 在 fd 的位置，bk 在 bk 的位置。

```c
#define bin_at(m, i) \
  (mbinptr) (((char *) &((m)->bins[((i) - 1) * 2]))			      \
             - offsetof (struct malloc_chunk, fd))   //减去 fd 在 malloc_chunk 中的偏移
```

main arena 的 malloc state 位于 libc 的数据段，所以读 unsorted chunk 的内容可以泄露 libc 基址，unsortedbins 这个双链的链表头在 main_arena 的 96 偏移处。

![image-20230205124431457](https://s2.loli.net/2023/02/05/MKRfPTmJQpcIZzC.png)

这个 unsortdchunk 长这样

![image-20230205130813093](https://s2.loli.net/2023/02/05/5Vzo3rls6kJuiBP.png)

main_arena + 96 长这样

![image-20230205131208627](https://s2.loli.net/2023/02/05/jdQqDTMm1iX62sV.png)



##### tcache_entry

tcache chunk 的基础结构。

```c
// 储存在 tcache 中的 chunk 的结构是 next + key
typedef struct tcache_entry
{
  //next 指示了该大小下一个 chunk 
  struct tcache_entry *next;
  //key 为 tcache_perthread_struct，记录了 tcache 所有链的信息，当首次 malloc 的时候会分配出来记录信息
  struct tcache_perthread_struct *key;
} tcache_entry;
```

通过 key 可以泄露 heap_base。

![image-20230205134747263](https://s2.loli.net/2023/02/05/UPjy6Yq5IQrt1RM.png)

##### tcache_perthread_struct

```c
//记录了 tcache 所有链的信息，当首次 malloc 的时候会分配出来记录信息
typedef struct tcache_perthread_struct
{
  //counts 记录每个 index 的 chunk 个数
  uint16_t counts[TCACHE_MAX_BINS];
  //entries 记录下一个将要 malloc 的 chunk 位置
  tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;
```

高版本 glibc 在 malloc 时的第一个大小 0x290 的 chunk 就是它，用来记录 tcache 的信息。

![image-20230205132219396](https://s2.loli.net/2023/02/05/jYNz1gKrZQ6TeJ5.png)

`__int_malloc` 里 `MAYBE_INIT_TCACHE ();`  会检查 tcache 是否初始化，如无则调用 `tcache_init()`，布置 tcache_perthread_struct。

```c
# define MAYBE_INIT_TCACHE() \
  if (__glibc_unlikely (tcache == NULL)) \
    tcache_init();
```

#### tcache 相关函数

malloc 过程 `__libc_malloc` 调用 `tcache_get`，free 过程 `_int_free` 调用 `tcache_put`。

##### tcache_get

`PROTECT_PTR` 和 `REVEAL_PTR` 函数和 safe_linking 有关，next 指针不是 “明文” 写地址，`PROTECT_PTR` 加密，`REVEAL_PTR` 解密。

```c
//调用 tcache_get 函数时必须保证 tc_idx 合理，并且有 chunk 的以取用
static __always_inline void *
tcache_get (size_t tc_idx)
{
  tcache_entry *e = tcache->entries[tc_idx];

  //检查是否对齐 MALLOC_ALIGN_MASK = 2 * SIZE_SZ -1 地址是 0x10 的倍数
  if (__glibc_unlikely (!aligned_OK (e)))
    malloc_printerr ("malloc(): unaligned tcache chunk detected");

  //与 PROTECT_PTR 的作用相反得到 next chunk
  tcache->entries[tc_idx] = REVEAL_PTR (e->next);
  --(tcache->counts[tc_idx]);

  //取消 tcache 标记
  e->key = NULL;
  return (void *) e;
}
```



##### tcache_put

```c
//调用 tcache_put 函数时必须保证 tc_idx 合理，并且有空间放入新的 chunk
static __always_inline void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
  //chunk2mem 指向 chunk 的 mem 部分
  tcache_entry *e = (tcache_entry *) chunk2mem (chunk);

  // 标记这个 chunk "in the tcache" 所以 _int_free 中的检查可以识别 double free
  e->key = tcache;

  //next 储存了 next变量地址和真正 next chunk 的地址运算的结果。因此若要更改 next 指针，需要知道堆地址。
  e->next = PROTECT_PTR (&e->next, tcache->entries[tc_idx]);

  //更新对应大小的 next chunk 和 个数
  tcache->entries[tc_idx] = e;
  ++(tcache->counts[tc_idx]);
}
```



##### safe_linking

glibc 2.32 引入 safe_linking

```c
 #define PROTECT_PTR(pos, ptr) \\
   ((__typeof (ptr)) ((((size_t) pos) >> 12) ^ ((size_t) ptr)))
 #define REVEAL_PTR(ptr)  PROTECT_PTR (&ptr, ptr)
```

gdb 里验证（先进后出）

```bash
pwndbg> bins
tcachebins
0xd0 [  4]: 0x55555555b510 —▸ 0x55555555b440 —▸ 0x55555555b370 —▸ 0x55555555b2a0 ◂— 0x0
pwndbg> x /10gx 0x55555555b510
0x55555555b510: 0x000055500000e11b      0x000055555555b010
0x55555555b520: 0x0000000000000000      0x0000000000000000
0x55555555b530: 0x0000000000000000      0x0000000000000000
0x55555555b540: 0x0000000000000000      0x0000000000000000
0x55555555b550: 0x0000000000000000      0x0000000000000000
>>> print((0x55555555b510 >> 12 ) ^ 0x000055500000e11b)
93824992261184
>>> hex(93824992261184)
'0x55555555b440'
```



##### __libc_malloc

`__libc_malloc` 函数是 malloc 过程的入口，在一开始就分配了 tcache，似乎轮不到 `_int_malloc` 做什么了，所以也不知道 `_int_malloc` 里的检查时什么用的。

```c
#if USE_TCACHE
  /* int_free also calls request2size, be careful to not pad twice.  */
  size_t tbytes;
  
  //检查申请地址的范围，并转换成对齐的地址储存在 tbytes 中
  if (!checked_request2size (bytes, &tbytes))
    {
      __set_errno (ENOMEM);
      return NULL;
    }
  //计算 tc index
  size_t tc_idx = csize2tidx (tbytes);

  //如果 tcache 没有初始化就初始化一下
  MAYBE_INIT_TCACHE ();

  //如果 1.大小在 tcache 的范围 2.tcache 初始化 3.tcache 有对应大小的 chunk，就拿出来  
  DIAG_PUSH_NEEDS_COMMENT;
  if (tc_idx < mp_.tcache_bins
      && tcache
      && tcache->counts[tc_idx] > 0)
    {
      return tcache_get (tc_idx);
    }
  DIAG_POP_NEEDS_COMMENT;
#endif
```



##### _int_malloc

遇不到？懒得注了



##### _int_free

```c
#if USE_TCACHE
  {
    size_t tc_idx = csize2tidx (size);
    //如果开启 tcache 而且符合 tcache 大小
    if (tcache != NULL && tc_idx < mp_.tcache_bins)
      {
    tcache_entry *e = (tcache_entry *) chunk2mem (p);

    //检查是否 double free，首先检查 key 有没有被标记，但是有很小的概率误判，所以继续往下看
    if (__glibc_unlikely (e->key == tcache))
      {
        tcache_entry *tmp;
        LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
        for (tmp = tcache->entries[tc_idx];
         tmp;
         //从最后一个进入的 chunk 依次往前检查
         tmp = REVEAL_PTR (tmp->next))
          {
        //如果不对齐
        if (__glibc_unlikely (!aligned_OK (tmp)))
          malloc_printerr ("free(): unaligned chunk detected in tcache 2");
        //如果 double free （按照 121 的顺序也不行）
        if (tmp == e)
          malloc_printerr ("free(): double free detected in tcache 2");
        //如果上面的检查都通过了，那 key 的值就是巧合
          }
      }

    if (tcache->counts[tc_idx] < mp_.tcache_count)
      {
        tcache_put (p, tc_idx);
        return;
      }
      }
  }
#endif
```



#### consolidate

##### fastbin to unsortedbin

申请 largebin 大小的 chunk 触发 consolidate（其他情况没遇到，鸽了）

```c
//__int_malloc
  else
    {
      idx = largebin_index (nb);
      //如果有 fastchunk，把 fastbins 中的 chunk 放到 unsorted bins中
      if (atomic_load_relaxed (&av->have_fastchunks))
        malloc_consolidate (av);
    }
```

`malloc_consolidate (av);` 整理 fastbin 到 unsortedbin。

```c
static void malloc_consolidate(mstate av)
{
  mfastbinptr*    fb;                 /* current fastbin being consolidated */
  mfastbinptr*    maxfb;              /* last fastbin (for loop control) */
  mchunkptr       p;                  /* current chunk being consolidated */
  mchunkptr       nextp;              /* next chunk to consolidate */
  mchunkptr       unsorted_bin;       /* bin header */
  mchunkptr       first_unsorted;     /* chunk to link to */

  /* These have same use as in free() */
  mchunkptr       nextchunk;
  INTERNAL_SIZE_T size;
  INTERNAL_SIZE_T nextsize;
  INTERNAL_SIZE_T prevsize;
  int             nextinuse;

  atomic_store_relaxed (&av->have_fastchunks, false);

  unsorted_bin = unsorted_chunks(av);

  /*
    Remove each chunk from fast bin and consolidate it, placing it
    then in unsorted bin. Among other reasons for doing this,
    placing in unsorted bin avoids needing to calculate actual bins
    until malloc is sure that chunks aren't immediately going to be
    reused anyway.
  */

  maxfb = &fastbin (av, NFASTBINS - 1);
  //从 index 为 0 开始找
  fb = &fastbin (av, 0);
  do {
      // atomic_exchange_acq 将 newvalue 存在 mem，并返回 old value
    p = atomic_exchange_acq (fb, NULL); //第一个循环增加 index
    if (p != 0) {
      do {  //第二个循环在同一个 index 里找下一个 chunk
	{
        //是否对齐
	  if (__glibc_unlikely (misaligned_chunk (p)))
	    malloc_printerr ("malloc_consolidate(): "
			     "unaligned fastbin chunk detected");
        //检查 index 是否符合 size 位
	  unsigned int idx = fastbin_index (chunksize (p));
	  if ((&fastbin (av, idx)) != fb)
	    malloc_printerr ("malloc_consolidate(): invalid chunk size");
	}

	check_inuse_chunk(av, p);
	nextp = REVEAL_PTR (p->fd);

	/* Slightly streamlined version of consolidation code in free() */
	size = chunksize (p);
	nextchunk = chunk_at_offset(p, size);
	nextsize = chunksize(nextchunk);

    //如果 pre_inuse 则触发 unlink
	if (!prev_inuse(p)) {
	  prevsize = prev_size (p);
	  size += prevsize;
	  p = chunk_at_offset(p, -((long) prevsize));
      //检查前一个相邻 chunk 的 size 是否符合 pre_size
	  if (__glibc_unlikely (chunksize(p) != prevsize))
	    malloc_printerr ("corrupted size vs. prev_size in fastbins");
	  unlink_chunk (av, p);
	}

    //如果不和 top 相邻
	if (nextchunk != av->top) {
	  nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

      //如果下一个相邻 chunk in use，就 unlink。
	  if (!nextinuse) {
	    size += nextsize;
	    unlink_chunk (av, nextchunk);
	  } else
	    clear_inuse_bit_at_offset(nextchunk, 0);

      //放进 unsorted bin 链里
	  first_unsorted = unsorted_bin->fd;
	  unsorted_bin->fd = p;
	  first_unsorted->bk = p;

	  if (!in_smallbin_range (size)) {
	    p->fd_nextsize = NULL;
	    p->bk_nextsize = NULL;
	  }

	  set_head(p, size | PREV_INUSE);
	  p->bk = unsorted_bin;
	  p->fd = first_unsorted;
	  set_foot(p, size);
	}

    //如果和 top 相邻就把这个 fast_chunk 和 top 合并
	else {
	  size += nextsize;
	  set_head(p, size | PREV_INUSE);
	  av->top = p;
	}

      } while ( (p = nextp) != 0);

    }
  } while (fb++ != maxfb);
}
```

##### unsortedbin to small & large bin

太长了，不想读，再说吧（