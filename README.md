用c语言实现了一个单块内存池分配，包括初始化、使用内存和释放内存。
***
# 代码解释
## 头文件和宏定义
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MEM_PAGE_SIZE 0x1000  //4k
```
这部分代码包含了必要的头文件和一个宏定义，MEM_PAGE_SIZE 定义了内存页的大小为 4KB。
## 内存池结构体
```c
typedef struct mempool_s
{
    int block_size; // block size:32 bytes
    int free_count; // count of free block:128

    void *mem; // mem pointer
    void *ptr; // free block pointer
} mempool_t;
```
+ mempool_t 结构定义了内存池的基本信息：
+ block_size：每个内存块的大小，这里是32字节。
+ free_count：可用的内存块数量，这里是根据总内存页大小和每个块大小计算出来的。
+ mem：指向分配的内存页的指针。
+ ptr：指向下一个可用内存块的指针。
## 内存池初始化函数
```c
int mempool_init(mempool_t *mp, size_t block_size)
{
    if (!mp)
        return -1;
    memset(mp, 0, sizeof(mempool_t)); // reset memory, prevent unknown error

    mp->block_size = block_size;
    mp->free_count = MEM_PAGE_SIZE / block_size;

    mp->mem = malloc(MEM_PAGE_SIZE);

    if (!mp->mem)
        return -1;
    mp->ptr = mp->mem; // initialize pointer

    char *ptr = mp->ptr;
    for (int i = 0; i < mp->free_count; ++i)
    {
        *(char **)ptr = ptr + block_size;
        ptr += block_size;
    }
    *(char **)ptr = NULL;
    return 0;
}
```
+ mempool_init 函数用于初始化内存池：
  +  通过 memset 将 mempool_t 结构清零。
  + 计算可用的内存块数量。
  + 分配一个大小为 MEM_PAGE_SIZE 的内存页。
  + 初始化 ptr 指针，使其指向第一个可用的内存块。
  + 设置内存块之间的链表关系。
## 内存分配函数
```c
void *_malloc(mempool_t *mp, size_t size)
{
    if (!mp || mp->free_count == 0)
        return NULL;
    if (mp->block_size < size)
        return NULL;

    void *ptr = mp->ptr;
    mp->ptr = *(char **)ptr;

    mp->free_count--;
    return ptr;
}
```
+ _malloc 函数用于从内存池中分配内存：
  + 检查输入参数和内存池状态。
  + 返回 ptr 指针，这是一个指向可用内存块的指针。
## 内存释放函数
```c
void _free(mempool_t *mp, void *ptr)
{
    if (!mp)
        return;
    *(char **)ptr = mp->ptr;
    mp->ptr = ptr;

    mp->free_count++;
    printf("_free\n");
}
```
+ _free 函数用于释放从内存池中分配的内存：
  + 将被释放的内存块添加回内存池的空闲列表。

## 主函数（测试代码）
```c
int main()
{
    mempool_t mp;

    mempool_init(&mp, 32); // init memory pool

    void *p1 = malloc(&mp,5);
    printf("p1->%p\n",p1);
    void *p2 = malloc(&mp,10);
    printf("p2->%p\n",p2);
    void *p3 = malloc(&mp,15);
    printf("p3->%p\n",p3);
    void *p4 = malloc(&mp,20);
    printf("p4->%p\n",p4);

    free(&mp,p1);
    printf("mp.ptr->%p\n",mp.ptr);
    free(&mp,p3);
    printf("mp.ptr->%p\n",mp.ptr);

    void *p5 = malloc(&mp,25);
    printf("p5->%p\n",p5);
    void *p6 = malloc(&mp,30);
    printf("p6->%p\n",p6);
    return 0;

    return 0;
}
```
在 main 函数中，我们首先初始化内存池，然后分配和释放几个内存块，以测试内存池的功能。
***
# 代码使用
在终端输入如下命令：
```bash
gcc -o mem ./mem.c
```
在当前目录生成mem可执行文件
代码执行结果如下：
```bash
humertank@ubuntu:~/Documents/mem$ ./mem
p1->0x582c965582a0
p2->0x582c965582c0
p3->0x582c965582e0
p4->0x582c96558300
_free
mp.ptr->0x582c965582a0
_free
mp.ptr->0x582c965582e0
p5->0x582c965582e0
p6->0x582c965582a0
```

# 存在的问题
**这个内存池存在一些问题：**
1. 在 _malloc 和 _free 函数中，当内存池为空或大小不足时，返回了 NULL，会导致无法确定发生了什么错误。可以添加更具描述性的错误消息或使用日志系统来记录错误。
2. 目前的内存池在释放内存块时只是简单地将其添加到空闲列表中，但没有真正地释放。如果想要完全释放内存，需要添加一个 mempool_destroy 函数来释放所有分配的内存。
