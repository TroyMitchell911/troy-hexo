---
title: openmpi[3] btl框架的self组件分析
date: 2025-09-08 17:44:55
tags:
  - openmpi
categories:
  - openmpi
---

在openmpi[2]中，已经分析了openmpi的模块化架构。对于本章节来说，上一章提到的framework对应的是btl;component对应的就是self; mca的意思就是mca模块🤣

## self组件的分层

在opal/mca/btl/self目录中，可以看到如下文件：

```
❯ tree
.
├── btl_self.c
├── btl_self_component.c
├── btl_self_component.lo
├── btl_self_frag.c
├── btl_self_frag.h
├── btl_self_frag.lo
├── btl_self.h
├── btl_self.lo
├── libmca_btl_self.la
├── Makefile
├── Makefile.am
├── Makefile.in
└── owner.txt
```

除去不需要关心的编译相关和注释相关，我们剩下了这三个内容：

```
❯ tree
.
├── btl_self.c
├── btl_self_component.c
├── btl_self_frag.c
├── btl_self_frag.h
```


- btl_frag: 与传输相关
- btl_self_component: 组件初始化及注册
- btl_self: 提供ops函数

## component

在btl_self_component.c中，一共包含了四个函数：

```
static int mca_btl_self_component_register(void);
static int mca_btl_self_component_open(void);
static int mca_btl_self_component_close(void);

/**
 * SELF module initialization.
 *
 * @param num_btls (OUT)                  Number of BTLs returned in BTL array.
 * @param enable_progress_threads (IN)    Flag indicating whether BTL is allowed to have progress
 * threads
 * @param enable_mpi_threads (IN)         Flag indicating whether BTL must support multilple
 * simultaneous invocations from different threads
 *
 */
static mca_btl_base_module_t **
mca_btl_self_component_init(int *num_btls, bool enable_progress_threads, bool enable_mpi_threads);
```

### 调用顺序和关系

这四个函数在 Open MPI 启动过程中按照特定顺序被 MCA 框架调用：

register → open → init → close

#### register

调用时机：MCA 框架加载组件时最先调用，用于注册组件参数。

主要职责：

注册组件到 MCA 变量系统
设置 free list 相关参数（free_list_num、free_list_max、free_list_inc）
配置 BTL 模块的性能参数（带宽、延迟、消息大小限制等）
设置 BTL 功能标志（RDMA、就地发送等）

#### open

调用时机：参数注册完成后，在组件初始化之前调用。

主要职责：

构造三个 opal_free_list_t 对象（eager、send、rdma fragments）
为后续的内存池初始化做准备
不进行实际的内存分配，只是对象构造

#### init

调用时机：组件打开后，当 BTL 框架需要实际创建 BTL 模块时调用。

主要职责：

初始化三个 free list，分配实际内存池
为不同类型的 fragment 设置不同的大小和对齐要求
创建并返回 BTL 模块数组给上层使用
这是唯一返回实际可用 BTL 模块的函数

#### close

调用时机：程序结束或组件卸载时调用，用于清理资源。

主要职责：

析构三个 opal_free_list_t 对象
释放在 open 阶段分配的资源
确保没有内存泄漏


### 关于构造的引入

在open/close中第一次见到了面向对象的真容

```
static int mca_btl_self_component_open(void)
{
    /* initialize objects */
    OBJ_CONSTRUCT(&mca_btl_self_component.self_frags_eager, opal_free_list_t);
    OBJ_CONSTRUCT(&mca_btl_self_component.self_frags_send, opal_free_list_t);
    OBJ_CONSTRUCT(&mca_btl_self_component.self_frags_rdma, opal_free_list_t);

    return OPAL_SUCCESS;
}

/*
 * component cleanup - sanity checking of queue lengths
 */

static int mca_btl_self_component_close(void)
{
    OBJ_DESTRUCT(&mca_btl_self_component.self_frags_eager);
    OBJ_DESTRUCT(&mca_btl_self_component.self_frags_send);
    OBJ_DESTRUCT(&mca_btl_self_component.self_frags_rdma);
    return OPAL_SUCCESS;
}
```

首先来看open函数中对opal_free_list_t这个类的构造宏是如何构成的:

```
#define OBJ_CONSTRUCT(object, type)                        \
    do {                                                   \
        OBJ_CONSTRUCT_INTERNAL((object), OBJ_CLASS(type)); \
    } while (0)
#define OBJ_CONSTRUCT_INTERNAL(object, type)                      \
    do {                                                          \
        OBJ_SET_MAGIC_ID((object), OPAL_OBJ_MAGIC_ID);            \
        if (opal_class_init_epoch != (type)->cls_initialized) {   \
            opal_class_initialize((type));                        \
        }                                                         \
        ((opal_object_t *) (object))->obj_class = (type);         \
        ((opal_object_t *) (object))->obj_reference_count = 1;    \
        opal_obj_run_constructors((opal_object_t *) (object));    \
        OBJ_REMEMBER_FILE_AND_LINENO(object, __FILE__, __LINE__); \
    } while (0)
```

OBJ_CONSTRUCT本质上就是调用OBJ_CONSTRUCT_INTERNAL.

在OBJ_CONSTRUCT_INTERNAL里面，先设置一个MAGIC_ID(如果没使能DEBUG的话，这就是nop)

之后来判断世代计数器，这里笔者也没有细究代码，不过来看是opal这个子系统初始化或者释放资源的时候会重新设置这个epoch的变量。在opal_class_initialize里面会初始化opal的class系统，然后设置epoch变量。

根据opal_class_initialize的注释也能看出:

```
/*
 * Lazy initialization of class descriptor.
 */
void opal_class_initialize(opal_class_t *cls)
```


思考：这里会带来空间占用的问题，如果调用OBJ_CONSTRUCT_INTERNAL的函数多了，这点懒加载带来的性能改变是否真的值得用这么多空间换吗？

即答: 几十KB而已🤣

之后会调用opal_obj_run_constructors函数来运行构造函数

```
/**
 * Run the hierarchy of class constructors for this object, in a
 * parent-first order.
 *
 * Do not use this function directly: use OBJ_CONSTRUCT() instead.
 *
 * WARNING: This implementation relies on a hardwired maximum depth of
 * the inheritance tree!!!
 *
 * Hardwired for fairly shallow inheritance trees
 * @param size          Pointer to the object.
 */
static inline void opal_obj_run_constructors(opal_object_t *object)
{
    opal_construct_t *cls_construct;

    assert(NULL != object->obj_class);

    cls_construct = object->obj_class->cls_construct_array;
    while (NULL != *cls_construct) {
        (*cls_construct)(object);
        cls_construct++;
    }
}
```

## frag

frag.c这个文件是为了抽象出self这个组件的传输相关内容。

根据self的源码来看，mca_btl_base_descriptor_t是btl进行传输的基础描述符。

frag相当于在这个基础描述符上增加了一些东西。

frag定义如下:

```
/**
 * shared memory send fragment derived type.
 */
struct mca_btl_self_frag_t {
    mca_btl_base_descriptor_t base;
    mca_btl_base_segment_t segments[2];
    struct mca_btl_base_endpoint_t *endpoint;
    opal_free_list_t *list;
    size_t size;
    unsigned char data[];
};
typedef struct mca_btl_self_frag_t mca_btl_self_frag_t;
typedef struct mca_btl_self_frag_t mca_btl_self_frag_eager_t;
typedef struct mca_btl_self_frag_t mca_btl_self_frag_send_t;
typedef struct mca_btl_self_frag_t mca_btl_self_frag_rdma_t;
```

由于openmpi采用了面向对象的思想，所以base这里可以看作是frag的父对象。

**segment**

这里面有一个segment需要额外注意，另外一个endpoint在self这个组件中没有用到。

segment定义如下:

```
/**
 * Describes a region/segment of memory that is addressable
 * by an BTL.
 *
 * Note: In many cases the alloc and prepare methods of BTLs
 * do not return a mca_btl_base_segment_t but instead return a
 * subclass. Extreme care should be used when modifying
 * BTL segments to prevent overwriting internal BTL data.
 *
 * All BTLs MUST use base segments when calling registered
 * Callbacks.
 *
 * BTL MUST use mca_btl_base_segment_t or a subclass and
 * MUST store their segment length in btl_seg_size. BTLs
 * MUST specify a segment no larger than MCA_BTL_SEG_MAX_SIZE.
 */

struct mca_btl_base_segment_t {
    /** Address of the memory */
    opal_ptr_t seg_addr;
    /** Length in bytes */
    uint64_t seg_len;
};
typedef struct mca_btl_base_segment_t mca_btl_base_segment_t;
```

每个 segment 包含两个关键字段：

- seg_addr：指向数据的内存地址
- seg_len：数据段的长度

frag这里又用到了我们在component中提到的类思想,在btl_self_frag.c中实现了类的构造函数：

```
static inline void mca_btl_self_frag_constructor(mca_btl_self_frag_t *frag)
{
    frag->base.des_flags = 0;
    frag->segments[0].seg_addr.pval = (void *) frag->data;
    frag->segments[0].seg_len = (uint32_t) frag->size;
    frag->base.des_segments = frag->segments;
    frag->base.des_segment_count = 1;
}

static void mca_btl_self_frag_eager_constructor(mca_btl_self_frag_t *frag)
{
    frag->list = &mca_btl_self_component.self_frags_eager;
    frag->size = mca_btl_self.btl_eager_limit;
    mca_btl_self_frag_constructor(frag);
}

static void mca_btl_self_frag_send_constructor(mca_btl_self_frag_t *frag)
{
    frag->list = &mca_btl_self_component.self_frags_send;
    frag->size = mca_btl_self.btl_max_send_size;
    mca_btl_self_frag_constructor(frag);
}

static void mca_btl_self_frag_rdma_constructor(mca_btl_self_frag_t *frag)
{
    frag->list = &mca_btl_self_component.self_frags_rdma;
    frag->size = MCA_BTL_SELF_MAX_INLINE_SIZE;
    mca_btl_self_frag_constructor(frag);
}

OBJ_CLASS_INSTANCE(mca_btl_self_frag_eager_t, mca_btl_base_descriptor_t,
                   mca_btl_self_frag_eager_constructor, NULL);

OBJ_CLASS_INSTANCE(mca_btl_self_frag_send_t, mca_btl_base_descriptor_t,
                   mca_btl_self_frag_send_constructor, NULL);

OBJ_CLASS_INSTANCE(mca_btl_self_frag_rdma_t, mca_btl_base_descriptor_t,
                   mca_btl_self_frag_rdma_constructor, NULL);
```

OBJ_CLASS_INSTANCE宏是定义类的实例，具体定义如下:

```
/**
 * Static initializer for a class descriptor
 *
 * @param NAME          Name of class
 * @param PARENT        Name of parent class
 * @param CONSTRUCTOR   Pointer to constructor
 * @param DESTRUCTOR    Pointer to destructor
 *
 * Put this in NAME.c
 */
#define OBJ_CLASS_INSTANCE(NAME, PARENT, CONSTRUCTOR, DESTRUCTOR) \
    opal_class_t NAME##_class = {#NAME,                           \
                                 OBJ_CLASS(PARENT),               \
                                 (opal_construct_t) CONSTRUCTOR,  \
                                 (opal_destruct_t) DESTRUCTOR,    \
                                 0,                               \
                                 0,                               \
                                 NULL,                            \
                                 NULL,                            \
                                 sizeof(NAME)}
```


拿frag_eager举例，frag_eager的定义在.h文件中为:

```
/**
 * shared memory send fragment derived type.
 */
struct mca_btl_self_frag_t {
    mca_btl_base_descriptor_t base;
    mca_btl_base_segment_t segments[2];
    struct mca_btl_base_endpoint_t *endpoint;
    opal_free_list_t *list;
    size_t size;
    unsigned char data[];
};
typedef struct mca_btl_self_frag_t mca_btl_self_frag_t;
typedef struct mca_btl_self_frag_t mca_btl_self_frag_eager_t;
```

可以看到frag_eager就是frag的alias而已，最终会被OBJ_CLASS_INSTANCE定义为mca_btl_self_frag_rdma_t_class的一个结构体，父对象是base也就是mca_btl_base_descriptor_t.

这很合理，因为frag结构体第一个成员就是base, 在openmpi中估计会通过强制类型转换来拿到父对象。

这里注意，并没有析构函数，这是因为并没有什么动态申请的资源好释放的。

在frag.h文件中进行了类的声明：

```
OBJ_CLASS_DECLARATION(mca_btl_self_frag_eager_t);
OBJ_CLASS_DECLARATION(mca_btl_self_frag_send_t);
OBJ_CLASS_DECLARATION(mca_btl_self_frag_rdma_t);
```

OBJ_CLASS_DECLARATION的定义如下:

```
/**
 * Declaration for class descriptor
 *
 * @param NAME          Name of class
 *
 * Put this in NAME.h
 */
#define OBJ_CLASS_DECLARATION(NAME) extern opal_class_t NAME##_class
```

继续拿frag_eager举例，其实就是extern mca_btl_self_frag_rdma_t_class这个结构体变量。

以便能够在其他文件中访问。这里openmpi框架会做好互斥访问的。

但是翻遍了self的源码树，也没发现在component里面显示调用的OBJ_CONSTRUCT宏，也就是没地方进入frag的构造函数。别急，接下来进入opal_free_list的“魔法”（其实就是隐藏在api里面了）


## opal_free_list

opal_free_list 是 Open MPI 中的一个高效内存管理机制，用于预分配和重用固定大小的内存块。

它基于 LIFO (Last-In-First-Out) 原理，提供快速的内存分配和释放操作。

在component的init函数中，我们实际初始化了opal_free_list, 是通过opal_free_list_init函数实现的。

opal_free_list_init函数定义如下：

```
/**
 * Initialize a free list.
 *
 * @param free_list                (IN)  Free list.
 * @param frag_size                (IN)  Size of each element - allocated by malloc.
 * @param frag_alignment           (IN)  Fragment alignment.
 * @param frag_class               (IN)  opal_class_t of element - used to initialize allocated
 * elements.
 * @param payload_buffer_size      (IN)  Size of payload buffer - allocated from mpool.
 * @param payload_buffer_alignment (IN)  Payload buffer alignment.
 * @param num_elements_to_alloc    (IN)  Initial number of elements to allocate.
 * @param max_elements_to_alloc    (IN)  Maximum number of elements to allocate.
 * @param num_elements_per_alloc   (IN)  Number of elements to grow by per allocation.
 * @param mpool                    (IN)  Optional memory pool for allocations.
 * @param rcache_reg_flags         (IN)  Flags to pass to rcache registration function.
 * @param rcache                   (IN)  Optional registration cache.
 * @param item_init                (IN)  Optional item initialization function
 * @param ctx                      (IN)  Initialization function context.
 */

OPAL_DECLSPEC int opal_free_list_init(opal_free_list_t *free_list, size_t frag_size,
                                      size_t frag_alignment, opal_class_t *frag_class,
                                      size_t payload_buffer_size, size_t payload_buffer_alignment,
                                      int num_elements_to_alloc, int max_elements_to_alloc,
                                      int num_elements_per_alloc,
                                      struct mca_mpool_base_module_t *mpool, int rcache_reg_flags,
                                      struct mca_rcache_base_module_t *rcache,
                                      opal_free_list_item_init_fn_t item_init, void *ctx);
```

这里函数参数比较复杂，我们逐个分析：

- 核心参数
  - frag_size：每个元素的大小，通过 malloc 分配
  - frag_alignment：Fragment 的内存对齐要求
  - frag_class：用于初始化分配元素的 opal_class_t 类型
  - payload_buffer_size：负载缓冲区大小，从 mpool 分配
  - payload_buffer_alignment：负载缓冲区的对齐要求
- 分配控制参数
  - num_elements_to_alloc：初始分配的元素数量
  - max_elements_to_alloc：最大可分配的元素数量
  - num_elements_per_alloc：每次增长时分配的元素数量
- 高级参数
  - mpool：可选的内存池，用于分配
  - rcache_reg_flags：传递给 rcache 注册函数的标志
  - rcache：可选的注册缓存
  - item_init：可选的元素初始化函数
  - ctx：初始化函数的上下文

回到component的调用中：

```
    ret = opal_free_list_init(&mca_btl_self_component.self_frags_eager,
                              sizeof(mca_btl_self_frag_eager_t) + mca_btl_self.btl_eager_limit,
                              opal_cache_line_size, OBJ_CLASS(mca_btl_self_frag_eager_t), 0,
                              opal_cache_line_size, mca_btl_self_component.free_list_num,
                              mca_btl_self_component.free_list_max,
                              mca_btl_self_component.free_list_inc, NULL, 0, NULL, NULL, NULL);
```


可以看到frag_size为frag_eager的大小 + eager_limit的大小，这是为什么呢？

在frag的定义中最后一个成员是`unsigned char data[];`, 所以这eager_limit是为这个data申请的内存。

另一个需要关心的内容是frag_class参数，这里直接调用了`OBJ_CLASS(mca_btl_self_frag_eager_t)`,这个宏在上面已经说过了。

### opal_free_list实现携带类的析构

opal_free_list只是一个数据结构，需要实现真正数据交互还需要携带一个类。

携带类指的就是这个意思。

在opal_free_list_init函数中会调用 opal_free_list_grow_st函数

#### opal_free_list_init

```
int opal_free_list_init(opal_free_list_t *flist, size_t frag_size, size_t frag_alignment,
                        opal_class_t *frag_class, size_t payload_buffer_size,
                        size_t payload_buffer_alignment, int num_elements_to_alloc,
                        int max_elements_to_alloc, int num_elements_per_alloc,
                        mca_mpool_base_module_t *mpool, int rcache_reg_flags,
                        mca_rcache_base_module_t *rcache, opal_free_list_item_init_fn_t item_init,
                        void *ctx)
{

    ...

    if (frag_class) {
        flist->fl_frag_class = frag_class;
    }

    ...

    if (num_elements_to_alloc) {
        return opal_free_list_grow_st(flist, num_elements_to_alloc, NULL);
    }

    return OPAL_SUCCESS;
}

```

其中调用` opal_free_list_grow_st`才是真正能够实现携带类构造函数的函数：

```
int opal_free_list_grow_st(opal_free_list_t *flist, size_t num_elements,
                           opal_free_list_item_t **item_out)
{

    ...
    for (size_t i = 0; i < num_elements; ++i) {
        opal_free_list_item_t *item = (opal_free_list_item_t *) ptr;
        item->registration = reg;
        item->ptr = payload_ptr;

        OBJ_CONSTRUCT_INTERNAL(item, flist->fl_frag_class);
```

这里很清晰了，在opal_free_list_grow_st函数中调用`OBJ_CONSTRUCT_INTERNAL`宏来进行携带类的构造。

所以携带类构造函数在以下两种情况被执行:

- 初始化时调用：当 opal_free_list_init 被调用且 num_elements_to_alloc > 0 时，会立即调用 opal_free_list_grow_st 来预分配元素：
- 动态增长时调用：当 free list 为空需要增长时，也会调用构造函数初始化新分配的元素。

#### opal_free_list_get

opal_free_list_get函数可以从list中拿到一个携带类资源。

#### opal_free_list_return

opal_free_list_return函数可以释放一个list中的携带类。

## module ops

接下来就是实现对于btl framework需要实现的ops了：

```
/* btl self module */
mca_btl_base_module_t mca_btl_self = {.btl_component = &mca_btl_self_component.super,
                                      .btl_add_procs = mca_btl_self_add_procs,
                                      .btl_del_procs = mca_btl_self_del_procs,
                                      .btl_finalize = mca_btl_self_finalize,
                                      .btl_alloc = mca_btl_self_alloc,
                                      .btl_free = mca_btl_self_free,
                                      .btl_prepare_src = mca_btl_self_prepare_src,
                                      .btl_send = mca_btl_self_send,
                                      .btl_sendi = mca_btl_self_sendi,
                                      .btl_put = mca_btl_self_put,
                                      .btl_get = mca_btl_self_get,
                                      .btl_dump = mca_btl_base_dump};
```

### btl_add_procs - 进程添加

BTL Self 只对自身进程设置可达性，通过比较进程名称来判断是否为同一进程。

add_procs负责：

- 进程可达性检测：确定哪些进程可以通过当前 BTL 模块到达
- endpoint 创建：为可达的进程创建通信端点
- 位图设置：在 reachable 位图中标记可达的进程
- 连接建立：建立与远程进程的实际连接（如 TCP 连接）

self的add_procs实现非常简单, 因为只需要处理本进程内：

```
/**
 * PML->BTL notification of change in the process list.
 * PML->BTL Notification that a receive fragment has been matched.
 * Called for message that is send from process with the virtual
 * address of the shared memory segment being different than that of
 * the receiver.
 *
 * @param btl (IN)
 * @param proc (IN)
 * @param peer (OUT)
 * @return     OPAL_SUCCESS or error status on failure.
 *
 */
static int mca_btl_self_add_procs(struct mca_btl_base_module_t *btl, size_t nprocs,
                                  struct opal_proc_t **procs,
                                  struct mca_btl_base_endpoint_t **peers,
                                  opal_bitmap_t *reachability)
{
    for (int i = 0; i < (int) nprocs; i++) {
        if (0 == opal_compare_proc(procs[i]->proc_name, OPAL_PROC_MY_NAME)) {
            opal_bitmap_set_bit(reachability, i);
            /* need to return something to keep the bml from ignoring us */
            peers[i] = (struct mca_btl_base_endpoint_t *) 1;
            break; /* there will always be only one ... */
        }
    }

    return OPAL_SUCCESS;
}
```

关键逻辑：

- 遍历所有传入的进程
- 通过 opal_compare_proc 比较进程名称与当前进程
- 如果是同一进程，设置位图并创建虚拟 endpoint
- 只会有一个进程匹配（自己）

#### bitmap

位图的作用: reachable 位图是一个关键的数据结构，用于：

- 标记可达性：每个位对应一个进程，设置为 1 表示该进程可通过此 BTL 到达
- BTL 选择：上层（BML）根据位图决定使用哪些 BTL 与特定进程通信
- 负载均衡：当多个 BTL 都能到达同一进程时，可以进行选择

### btl_del_procs - 进程删除

用于清理进程相关资源, self的实现很简单，直接返回成功，因为没有需要清理的资源。

```
/**
 * PML->BTL notification of change in the process list.
 *
 * @param btl (IN)     BTL instance
 * @param proc (IN)    Peer process
 * @param peer (IN)    Peer addressing information.
 * @return             Status indicating if cleanup was successful
 *
 */
static int mca_btl_self_del_procs(struct mca_btl_base_module_t *btl, size_t nprocs,
                                  struct opal_proc_t **procs,
                                  struct mca_btl_base_endpoint_t **peers)
{
    return OPAL_SUCCESS;
}
```

### btl_alloc - 内存分配

self实现如下:

```
/**
 * Allocate a segment.
 *
 * @param btl (IN)      BTL module
 * @param size (IN)     Request segment size.
 */
static mca_btl_base_descriptor_t *mca_btl_self_alloc(struct mca_btl_base_module_t *btl,
                                                     struct mca_btl_base_endpoint_t *endpoint,
                                                     uint8_t order, size_t size, uint32_t flags)
{
    mca_btl_self_frag_t *frag = NULL;

    if (size <= MCA_BTL_SELF_MAX_INLINE_SIZE) {
        MCA_BTL_SELF_FRAG_ALLOC_RDMA(frag);
    } else if (size <= mca_btl_self.btl_eager_limit) {
        MCA_BTL_SELF_FRAG_ALLOC_EAGER(frag);
    } else if (size <= btl->btl_max_send_size) {
        MCA_BTL_SELF_FRAG_ALLOC_SEND(frag);
    }

    if (OPAL_UNLIKELY(NULL == frag)) {
        return NULL;
    }

    frag->segments[0].seg_len = size;
    frag->base.des_segment_count = 1;
    frag->base.des_flags = flags;

    return &frag->base;
}
```

可以看到根据内存大小进行不同的选择：

- <= MCA_BTL_SELF_MAX_INLINE_SIZE -> rdma_frag
- <= mca_btl_self.btl_eager_limit -> eager_frag
- <= btl_max_send_size -> send_frag

### btl_prepare_src - 发送准备

prepare_src 可以根据数据特性选择最优的处理方式。比如对于连续数据可以避免不必要的内存拷贝。

这点我们会在下面的代码看到：

```
/**
 * Prepare data for send
 *
 * @param btl (IN)      BTL module
 */
static struct mca_btl_base_descriptor_t *mca_btl_self_prepare_src(
    struct mca_btl_base_module_t *btl, struct mca_btl_base_endpoint_t *endpoint,
    struct opal_convertor_t *convertor, uint8_t order, size_t reserve, size_t *size, uint32_t flags)
{
    bool inline_send = !(opal_convertor_need_buffers(convertor) || opal_convertor_on_device(convertor));
    size_t buffer_len = reserve + (inline_send ? 0 : *size);
    mca_btl_self_frag_t *frag;

    frag = (mca_btl_self_frag_t *) mca_btl_self_alloc(btl, endpoint, order, buffer_len, flags);
    if (OPAL_UNLIKELY(NULL == frag)) {
        return NULL;
    }

    ...

}
```


首先先获取是否需要内联发送，内联发送的条件是:

- opal_convertor_need_buffers = 0, 也就是说数据是连续的，不需要额外缓冲区进行打包
- opal_convertor_on_device = 0, 数据没有在设备上，比如GPU

这里又引入了一个新的数据结构convertor, 它其实就是数据buffer的一个封装，主要作用：

- 处理非连续数据类型：将复杂的 MPI 数据类型（如结构体、向量等）转换为可传输的连续字节流
- 数据打包/解包：通过 opal_convertor_pack 和 opal_convertor_unpack 函数进行数据转换
- 跨架构支持：处理不同架构间的数据格式转换

在buffer_len中比较迷惑的一点就是reserve, 预留内存大小。是上层协议（如 PML）需要在数据前面预留的头部空间，用于存放协议头信息

接下来申请一个frag，之后就会遇到两个路径。

这里需要先提前说一下segments的规范：

- 当为连续数据的时候， segments[0]为PML层的头部数据保留， segments[1]指向实际数据的指针（零拷贝）
- 当为非连续数据的时候， segments[0]为PML层的头部数据+实际数据， segments[1] 没有作用。

#### 连续数据内联准备发送

```
    } else {
        void *data_ptr;

        opal_convertor_get_current_pointer(convertor, &data_ptr);

        frag->segments[1].seg_addr.pval = data_ptr;
        frag->segments[1].seg_len = *size;
        frag->base.des_segment_count = 2;
    }
    
    return &frag->base;
```

当inline_send = true的时候，就走的else也就是内联发送分支。

从UNLIKELY这个宏可以看出，绝大部分时间应该走的是这个分支。

在内联发送中，将convertor的当前指针拿到data_prt变量中，然后赋值给segments[1]。

最后直接返回了frag的父对象也就是base描述符的指针。

这样接收方可以直接从convertor指向的指针中直接拿数据，0拷贝。

#### 非连续数据或非本地数据（在设备上）准备发送

```
    /* non-contiguous data */
    if (OPAL_UNLIKELY(!inline_send)) {
        struct iovec iov = {.iov_len = *size,
                            .iov_base = (IOVBASE_TYPE *) ((uintptr_t) frag->data + reserve)};
        size_t max_data = *size;
        uint32_t iov_count = 1;
        int rc;

        rc = opal_convertor_pack(convertor, &iov, &iov_count, &max_data);
        if (rc < 0) {
            mca_btl_self_free(btl, &frag->base);
            return NULL;
        }

        *size = max_data;
        frag->segments[0].seg_len = reserve + max_data;
```

当inline_send = false的时候，就走的if分支。

此时需要创建一个iovec的数据结构，iovec 是一个标准的 I/O 向量结构，包含 iov_base（指针）和 iov_len（长度）两个字段。在处理非连续数据时，它用于指定 opal_convertor_pack 函数将数据打包到哪个缓冲区位置。

这里的iovec的赋值更加确定了我们之前解释的segments.

可以看到iovec的iov_base的地址跳过了reserve, 充分说明是给PML层的头部预留的空间。

之后通过`opal_convertor_pack`函数将数据打包到iovec指向的地址中。

最后更新segments[0].seg_len。

这里可能比较疑惑seg的addr怎么没有像连续数据那样设置，因为在frag的构造函数中已经指向了frag->data了，而上面iovec拷贝的目标地址正是frag->data.

### btl_send - 消息发送

```
/**
 * Initiate a send to the peer.
 *
 * @param btl (IN)      BTL module
 * @param peer (IN)     BTL peer addressing
 */

static int mca_btl_self_send(struct mca_btl_base_module_t *btl,
                             struct mca_btl_base_endpoint_t *endpoint,
                             struct mca_btl_base_descriptor_t *des, mca_btl_base_tag_t tag)
{
    mca_btl_active_message_callback_t *reg = mca_btl_base_active_message_trigger + tag;
    mca_btl_base_receive_descriptor_t recv_desc = {.endpoint = endpoint,
                                                   .des_segments = des->des_segments,
                                                   .des_segment_count = des->des_segment_count,
                                                   .tag = tag,
                                                   .cbdata = reg->cbdata};
    int btl_ownership = (des->des_flags & MCA_BTL_DES_FLAGS_BTL_OWNERSHIP);

    /* upcall */
    reg->cbfunc(btl, &recv_desc);

    /* send completion */
    if (des->des_flags & MCA_BTL_DES_SEND_ALWAYS_CALLBACK) {
        des->des_cbfunc(btl, endpoint, des, OPAL_SUCCESS);
    }
    if (btl_ownership) {
        mca_btl_self_free(btl, des);
    }
    return 1;
}
```

在send函数中，由于self组件是同线程内使用，所以不需要任何的传送机制。

直接调用回调函数模拟接收就可以了。

MCA_BTL_DES_SEND_ALWAYS_CALLBACK宏用来判断是否需要调用des的回调。

btl_ownership用来判断是否需要由btl层释放描述符。

### btl_sendi - 立即发送

sendi函数是指的立即发送，也就是将prepare_src和send函数合并到一步"原子性"的完成。

先看一下函数的定义：

```
static int mca_btl_self_sendi(struct mca_btl_base_module_t *btl,
                              struct mca_btl_base_endpoint_t *endpoint,
                              struct opal_convertor_t *convertor, void *header, size_t header_size,
                              size_t payload_size, uint8_t order, uint32_t flags,
                              mca_btl_base_tag_t tag, mca_btl_base_descriptor_t **descriptor)
```

这里需要注意几个参数：

- payload_size: 负载数据大小
- header_size: 协议头大小

endpoint不需要注意，在add_procs里面我们设置成-1了也就是没用到这东西

在sendi的开头，先是进行了如下操作：

```
    if (!payload_size ||
        !(opal_convertor_need_buffers(convertor) ||
          opal_convertor_on_device(convertor))) {
        void *data_ptr = NULL;
        if (payload_size) {
            opal_convertor_get_current_pointer(convertor, &data_ptr);
        }

        mca_btl_base_segment_t segments[2] = {{.seg_addr.pval = header, .seg_len = header_size},
                                              {.seg_addr.pval = data_ptr, .seg_len = payload_size}};
        mca_btl_base_descriptor_t des = {.des_segments = segments,
                                         .des_segment_count = payload_size ? 2 : 1,
                                         .des_flags = 0};

        (void) mca_btl_self_send(btl, endpoint, &des, tag);
        return OPAL_SUCCESS;
    }
```

这一条路径就是快速路径，可以看到是直接用的临时变量des和segments，而不是用的alloc的frag.

因为调用 mca_btl_self_alloc 需要从内存池中分配片段，涉及内存池操作和可能的锁竞争。

如果负载数据大小为0 || (是连续的数据 && 数据不在设备上)，那么就执行这个if函数。

人话：负载数据大小为0或者内联发送

进入这个if的时候会再次判断是否payload_size是否不为0,如果不为0，就获取convertor的指针到data_ptr中。

封装进入segments, 设置协议头地址和大小/数据地址和大小。

最后将segments给到des, 然后直接调用send函数。

如果没有成功进入快速路径的话，就走的如下代码：

```
    frag = mca_btl_self_prepare_src(btl, endpoint, convertor, order, header_size, &payload_size,
                                    flags | MCA_BTL_DES_FLAGS_BTL_OWNERSHIP);
    if (NULL == frag) {
        if( NULL != descriptor ) {
            *descriptor = NULL;
        }
        return OPAL_ERR_OUT_OF_RESOURCE;
    }

    memcpy(frag->des_segments[0].seg_addr.pval, header, header_size);
    (void) mca_btl_self_send(btl, endpoint, frag, tag);
    return OPAL_SUCCESS;
```


慢速路径其实就是将prepare_src和send函数合并到一起并没有做性能优化，唯一需要注意的就是通过`memcpy`函数获得了协议头的数据。

### btl_put/btl_get - RDMA put/get操作

self组件对于rdma的实现相当简单粗暴，因为都是同一个进程内的内存，所以直接可以memcpy

```
static int mca_btl_self_put(mca_btl_base_module_t *btl, struct mca_btl_base_endpoint_t *endpoint,
                            void *local_address, uint64_t remote_address,
                            mca_btl_base_registration_handle_t *local_handle,
                            mca_btl_base_registration_handle_t *remote_handle, size_t size,
                            int flags, int order, mca_btl_base_rdma_completion_fn_t cbfunc,
                            void *cbcontext, void *cbdata)
{
    memcpy((void *) (intptr_t) remote_address, local_address, size);

    cbfunc(btl, endpoint, local_address, NULL, cbcontext, cbdata, OPAL_SUCCESS);

    return OPAL_SUCCESS;
}

static int mca_btl_self_get(mca_btl_base_module_t *btl, struct mca_btl_base_endpoint_t *endpoint,
                            void *local_address, uint64_t remote_address,
                            mca_btl_base_registration_handle_t *local_handle,
                            mca_btl_base_registration_handle_t *remote_handle, size_t size,
                            int flags, int order, mca_btl_base_rdma_completion_fn_t cbfunc,
                            void *cbcontext, void *cbdata)
{
    memcpy(local_address, (void *) (intptr_t) remote_address, size);

    cbfunc(btl, endpoint, local_address, NULL, cbcontext, cbdata, OPAL_SUCCESS);

    return OPAL_SUCCESS;
}
```
