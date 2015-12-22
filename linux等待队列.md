等待队列在linux内核中有着举足轻重的作用，很多linux驱动都或多或少涉及到了等待队列。因此，对于linux内核及驱动开发者来说，掌握等待队列是必须课之一。
Linux内核的等待队列是以双循环链表为基础数据结构，与进程调度机制紧密结合，能够用于实现核心的异步事件通知机制。它有两种数据结构：等待队列头（`wait_queue_head_t`）和等待队列项（`wait_queue_t`，`typedef struct
__wait_queue wait_queue_t;`）。等待队列头和等待队列项中都包含一个list_head类型的域作为”连接件”。它通过一个双链表和把等待task的头，和等待的进程列表链接起来。下面具体介绍。

## 一.数据结构定义
### 列表头
文件路径：/include/linux/wait.h
```c++
struct __wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};

typedef struct __wait_queue_head wait_queue_head_t;//列表头

```

### 列表项

```c++
struct __wait_queue {
   unsigned int flags;
   void* private;
   wait_queue_func_t func;
   struct list_head task_list;
};

```

### 双向链表结构

```c++
struct list_head {
       struct list_head* next;
        struct list_head* prev;
};

```

## 二.作用
在内核里面，等待队列是有很多用处的，尤其是在中断处理、进程同步、定时等场合。可以使用等待队列在实现阻塞进程的唤醒。它以队列为基础数据结构，与进程调度机制紧密结合，能够用于实现内核中的异步事件通知机制，同步对系统资源的访问等。

## 三.字段详解

### 1、spinlock_t lock;
在对task_list与操作的过程中，使用该锁实现对等待队列的互斥访问。
### 2、srtuct list_head_t task_list;
双向循环链表，存放等待的进程。

## 操作
### 1.定义并初始化列表头
#### 直接定义
```c++
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
```
`init_waitqueue_head()`函数会将自旋锁初始化为未锁，等待队列初始化为空的双向循环链表。


```java
/include/linux/wait.h

#define init_waitqueue_head(q)                             
   do {                                
           static struct lock_class_key (__key);                                                         
           __init_waitqueue_head((q), (&__key));   
   } while (0)

```
```java
/kernel/wait.c

void __init_waitqueue_head(wait_queue_head_t *q, struct lock_class_key *key)
{
  spin_lock_init(&q->lock);
  lockdep_set_class(&q->lock, key);
  INIT_LIST_HEAD(&q->task_list);
}

```
```java
/include/linux/list.h

static inline void INIT_LIST_HEAD(struct list_head *list)
{
  list->next = list;
  list->prev = list;
}

```

#### 使用宏

`DECLARE_WAIT_QUEUE_HEAD(my_queue);`
```java
#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {                                
   .lock            = __SPIN_LOCK_UNLOCKED(name.lock),           
   .task_list       = { &(name).task_list, &(name).task_list } }

#define DECLARE_WAIT_QUEUE_HEAD(name)
   wait_queue_head_t name = __WAIT_QUEUE_HEAD_INITIALIZER(name)

```
定义并初始化LIST，相当于(1)

### 定义等待队列

`DECLARE_WAITQUEUE(name,tsk);`

```java
#define DECLARE_WAITQUEUE(name, tsk)                                      
   wait_queue_t name = __WAITQUEUE_INITIALIZER(name, tsk)

#define __WAITQUEUE_INITIALIZER(name, tsk) {                          
   .private   = tsk,                                         
   .func         = default_wake_function,                     
   .task_list       = { NULL, NULL } }
```
注意此处是定义一个wait_queue_t类型的变量name，并将其private设置为tsk，wait_queue_t 结构：

```java
struct __wait_queue {
   unsigned int flags;//指明该等待的进程是互斥进程还是非互斥进程，0是非互斥进程
   void *private;
   wait_queue_func_t func;
   struct list_head task_list;
};
```
其中flags域指明该等待的进程是互斥进程还是非互斥进程。其中0是非互斥进程，WQ_FLAG_EXCLUSIVE(0x01)是互斥进程。
等待队列(wait_queue_t)和等待对列头(wait_queue_head_t)的区别是等待队列是等待队列头的成员。
也就是说等待队列头的task_list域链接的成员就是等待队列类型的(wait_queue_t)。

### (从等待队列头中)添加／移出等待队列

#### 添加

设置等待的进程为非互斥进程，并将其添加进等待队列头(q)的队头中（list.h）

```java
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
  unsigned long flags;
  wait->flags &= ~WQ_FLAG_EXCLUSIVE;
  spin_lock_irqsave(&q->lock, flags);
  __add_wait_queue(q, wait);
  spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(add_wait_queue);
```
```
static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
{
        list_add(&new->task_list, &head->task_list);
}
```

```
/include/linux/list.h
static inline void list_add(struct list_head *new, struct list_head *head)
{
   __list_add(new, head, head->next);
}
```

```
/include/linux/list.h
static inline void __list_add(struct list_head *new,struct list_head *prev,struct list_head *next)
   next->prev = new;
   new->next = next;
   new->prev = prev;
   prev->next = new;
}
```

该函数也和add_wait_queue()函数功能基本一样，只不过它是将等待的进程(wait)设置为互斥进程，且添加到末尾

```
void add_wait_queue_exclusive(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;

    wait->flags |= WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&q->lock, flags);
    __add_wait_queue_tail(q, wait);
    spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(add_wait_queue_exclusive);
static inline void __add_wait_queue_tail(wait_queue_head_t *head,
                                         wait_queue_t *new)
{
     list_add_tail(&new->task_list, &head->task_list);
}

```

#### 移除
在等待的资源或事件满足时，进程被唤醒，使用该函数被从等待头中删除

```
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;

    spin_lock_irqsave(&q->lock, flags);
    __remove_wait_queue(q, wait);
    spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(remove_wait_queue);
static inline void __remove_wait_queue(wait_queue_head_t *head,wait_queue_t *old)
{
        list_del(&old->task_list);
}

```

## 等待事件

### wait_event()宏：

```java
/**
 * wait_event - sleep until a condition gets true
 * @wq: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 *
 * The process is put to sleep (TASK_UNINTERRUPTIBLE) until the
 * @condition evaluates to true. The @condition is checked each time
 * the waitqueue @wq is woken up.
 *
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 */
#define wait_event(wq, condition)                   \
do {                                    \
    if (condition)                          \
        break;                          \
    __wait_event(wq, condition);                    \
} while (0)
#define __wait_event(wq, condition)                                       \
  (void)___wait_event(wq, condition, TASK_UNINTERRUPTIBLE, 0, 0, schedule())
```

```java

/*
 * The below macro ___wait_event() has an explicit shadow of the __ret
 * variable when used from the wait_event_*() macros.
 *
 * This is so that both can use the ___wait_cond_timeout() construct
 * to wrap the condition.
 *
 * The type inconsistency of the wait_event_*() __ret variable is also
 * on purpose; we use long where we can return timeout values and int
 * otherwise.
 */

#define ___wait_event(wq, condition, state, exclusive, ret, cmd)      \
({                                                                     \
  __label__ __out;                                          \
  wait_queue_t __wait;//列表项                                             \
  long __ret = ret;        /* explicit shadow */                 \
                                                                  \
  INIT_LIST_HEAD(&__wait.task_list);                         \
  if (exclusive)                                                      \
          __wait.flags = WQ_FLAG_EXCLUSIVE;                  \
  else                                                             \
          __wait.flags = 0;                                    \
                                                                       \
  for (;;) {                                                       \
          long __int = prepare_to_wait_event(&wq, &__wait, state);\
                                                                  \
          if (condition)                                              \
                  break;                                           \
                                                                  \
          if (___wait_is_interruptible(state) && __int) {               \
                  __ret = __int;                                        \
                  if (exclusive) {                            \
                               abort_exclusive_wait(&wq, &__wait,      \
                                               state, NULL);     \
                          goto __out;                             \
                  }                                               \
                  break;                                           \
          }                                                       \
                                                                  \
          cmd;                                                     \
  }                                                               \
  finish_wait(&wq, &__wait);                                       \
__out:        __ret;                                                         \
})
```
在等待会列中睡眠直到condition为真。在等待的期间，进程会被置为TASK_UNINTERRUPTIBLE进入睡眠，直到condition变量变为真。每次进程被唤醒的时候都会检查
condition的值.
