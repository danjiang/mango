---
title: iOS 基础 - 使用互斥锁和条件变量实现读写锁
author: 但江
avatar: danjiang
location: 深圳 
category: programming
tag: ios-basic
---

Posix Thread 中已经实现了读写锁 pthread_rwlock_t，这里讲解的是如何通过互斥锁 pthread_mutex_t 和条件变量 pthread_cond_t 实现读写锁，对 iOS 中通过 NSLock 和 NSCondition 来实现读写锁起到一定的参考作用。

![Apple Develop Xcode](/images/apple-develop-xcode.jpg)

## 读写锁的数据结构

我们的 **pthread_rwlock_t** 数据类型含有一个互斥锁、两个条件变量、一个标志及三个计数器。我们将从接下来给出的函数中看出所有这些成员的用途。无论何时检查或操纵该结构，我们都必须持有其中的互斥锁成员 rw_mutex。该结构初始化成功后，标志成员 rw_magic 就被设置成 RW_MAGIC。所有函数都测试该成员，以检查调用者是否向某个已初始化的读写锁传递了指针。该读写锁被摧毁时，这个成员就被置为 0。注意计数器成员之一 rw_refcount 总是指示着本读写锁的当前状态：-1 表示它是一个写入锁（任意时刻这样的锁只能有一个），0 表示它是可用的，大于 0 的值则意味着它当前容纳着那么多的读出锁。

{% highlight c %}
/* include rwlockh */
#ifndef	__pthread_rwlock_h
#define	__pthread_rwlock_h

typedef struct {
  pthread_mutex_t rw_mutex; /* basic lock on this struct */
  pthread_cond_t rw_condreaders; /* for reader threads waiting */
  pthread_cond_t rw_condwriters; /* for writer threads waiting */
  int rw_magic; /* for error checking */
  int rw_nwaitreaders; /* the number waiting */
  int rw_nwaitwriters; /* the number waiting */
  int rw_refcount;
/* 4-1 if writer has the lock, else # readers holding the lock */
} pthread_rwlock_t;

#define	RW_MAGIC 0x19283746

/* 4following must have same order as elements in struct above */
#define	PTHREAD_RWLOCK_INITIALIZER { PTHREAD_MUTEX_INITIALIZER, \
			PTHREAD_COND_INITIALIZER, PTHREAD_COND_INITIALIZER, \
			RW_MAGIC, 0, 0, 0 }

typedef	int pthread_rwlockattr_t; /* dummy; not supported */

/* 4function prototypes */
int pthread_rwlock_destroy(pthread_rwlock_t *);
int pthread_rwlock_init(pthread_rwlock_t *, pthread_rwlockattr_t *);
int pthread_rwlock_rdlock(pthread_rwlock_t *);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *);
int pthread_rwlock_trywrlock(pthread_rwlock_t *);
int pthread_rwlock_unlock(pthread_rwlock_t *);
int pthread_rwlock_wrlock(pthread_rwlock_t *);
/* $$.bp$$ */
/* 4and our wrapper functions */
void Pthread_rwlock_destroy(pthread_rwlock_t *);
void Pthread_rwlock_init(pthread_rwlock_t *, pthread_rwlockattr_t *);
void Pthread_rwlock_rdlock(pthread_rwlock_t *);
int Pthread_rwlock_tryrdlock(pthread_rwlock_t *);
int Pthread_rwlock_trywrlock(pthread_rwlock_t *);
void Pthread_rwlock_unlock(pthread_rwlock_t *);
void Pthread_rwlock_wrlock(pthread_rwlock_t *);

#endif	__pthread_rwlock_h
/* end rwlockh */
{% endhighlight %}

## 读写锁的初始化

**pthread_rwlock_init** 动态初始化一个读写锁。初始化由调用者指定其指针的读写锁结构中的互斥锁和两个条件变量成员。所有三个计数器成员都设置为 0，rw_magic 成员则设置为表示该结构已初始化完毕的值。如果互斥锁或条件变量的初始化失败，那么小心地确保摧毁已初始化的对象，然后返回一个错误。

{% highlight c %}
/* include init */
#include "unpipc.h"
#include "pthread_rwlock.h"

int pthread_rwlock_init(pthread_rwlock_t *rw, pthread_rwlockattr_t *attr)
{
 int result;

 if (attr != NULL)
  return(EINVAL); /* not supported */

 if ((result = pthread_mutex_init(&rw->rw_mutex, NULL)) != 0)
  goto err1;
 if ((result = pthread_cond_init(&rw->rw_condreaders, NULL)) != 0)
  goto err2;
 if ((result = pthread_cond_init(&rw->rw_condwriters, NULL)) != 0)
  goto err3;
 rw->rw_nwaitreaders = 0;
 rw->rw_nwaitwriters = 0;
 rw->rw_refcount = 0;
 rw->rw_magic = RW_MAGIC;

 return(0);

 err3:
  pthread_cond_destroy(&rw->rw_condreaders);
 err2:
  pthread_mutex_destroy(&rw->rw_mutex);
 err1:
  return(result); /* an errno value */
}
/* end init */

void Pthread_rwlock_init(pthread_rwlock_t *rw, pthread_rwlockattr_t *attr)
{
 int n;

 if ((n = pthread_rwlock_init(rw, attr)) == 0)
  return;
 errno = n;
 err_sys("pthread_rwlock_init error");
}
{% endhighlight %}

## 读写锁的销毁

**pthread_rwlock_destroy** 在所有线程（包括调用者在内）都不再持有也不试图持有某个读写锁的时候摧毁该锁。首先检查由调用者指定的读写锁已不在使用中，然后给其中的互斥锁和两个条件变量成员调用合适的摧毁函数。

{% highlight c %}
#include "unpipc.h"
#include "pthread_rwlock.h"

int pthread_rwlock_destroy(pthread_rwlock_t *rw)
{
 if (rw->rw_magic != RW_MAGIC)
  return(EINVAL);
 if (rw->rw_refcount != 0 || rw->rw_nwaitreaders != 0 || rw->rw_nwaitwriters != 0)
  return(EBUSY);

 pthread_mutex_destroy(&rw->rw_mutex);
 pthread_cond_destroy(&rw->rw_condreaders);
 pthread_cond_destroy(&rw->rw_condwriters);
 rw->rw_magic = 0;

 return(0);
}
/* end destroy */

void Pthread_rwlock_destroy(pthread_rwlock_t *rw)
{
 int n;

 if ((n = pthread_rwlock_destroy(rw)) == 0)
  return;
 errno = n;
 err_sys("pthread_rwlock_destroy error");
}
{% endhighlight %}

## 读写锁的读取锁

**pthread_rwlock_rdlock** 的实现，无论何时操作 pthread_rwlock_t 类型的结构，都必须给其中的 rw_mutex 成员上锁。如果（a）rw_refcount 小于 0（意味着当前有一个写入者持有由调用者指定的读写锁），或者（b）有线程正等着获取该读写锁的一个写入锁（rw_nwaitwriters 大于 0），那么我们无法获取该读写锁的一个读出锁。如果这两个条件中有一个为真，我们就把rw_nwaitreaders 加 1。并在 rw_condreaders 条件变量上调用 pthread_cond_wait。我们稍后将看到，当给一个读写锁解锁时，首先检查是否有任何等待着的写入者，若没有则检查是否有任何等待着的读出者。如果有读出者在等待，那就向rw_condreaders 条件变量广播信号。取得读出锁后把 rw_refcount 加 1。互斥锁旋即释放。

{% highlight c %}
/* include rdlock */
#include "unpipc.h"
#include "pthread_rwlock.h"

int pthread_rwlock_rdlock(pthread_rwlock_t *rw)
{
 int result;

 if (rw->rw_magic != RW_MAGIC)
  return(EINVAL);

 if ((result = pthread_mutex_lock(&rw->rw_mutex)) != 0)
  return(result);

 /* 4give preference to waiting writers */
 while (rw->rw_refcount < 0 || rw->rw_nwaitwriters > 0) {
  rw->rw_nwaitreaders++;
  result = pthread_cond_wait(&rw->rw_condreaders, &rw->rw_mutex);
  rw->rw_nwaitreaders--;
  if (result != 0)
   break;
 }
 if (result == 0)
  rw->rw_refcount++; /* another reader has a read lock */

 pthread_mutex_unlock(&rw->rw_mutex);
 return (result);
}
/* end rdlock */

void Pthread_rwlock_rdlock(pthread_rwlock_t *rw)
{
 int n;

 if ((n = pthread_rwlock_rdlock(rw)) == 0)
  return;
 errno = n;
 err_sys("pthread_rwlock_rdlock error");
}
{% endhighlight %}

## 读写锁的非阻塞读取锁

**pthread_rwlock_tryrdlock** 在尝试获取一个读出锁时并不阻塞。如果当前有一个写入者持有调用者指定的读写锁，或者有线程在等待该读写锁的一个写入锁，那就返回 EBUSY 错误。否则，通过把 rw_refcount 加 1 获取该读写锁。

{% highlight c %}
/* include tryrdlock */
#include "unpipc.h"
#include "pthread_rwlock.h"

int pthread_rwlock_tryrdlock(pthread_rwlock_t *rw)
{
 int result;

 if (rw->rw_magic != RW_MAGIC)
  return(EINVAL);

 if ((result = pthread_mutex_lock(&rw->rw_mutex)) != 0)
  return(result);

 if (rw->rw_refcount < 0 || rw->rw_nwaitwriters > 0)
  result = EBUSY; /* held by a writer or waiting writers */
 else
  rw->rw_refcount++; /* increment count of reader locks */

 pthread_mutex_unlock(&rw->rw_mutex);
 return(result);
}
/* end tryrdlock */

int Pthread_rwlock_tryrdlock(pthread_rwlock_t *rw)
{
 int n;

 if ((n = pthread_rwlock_tryrdlock(rw)) != 0) {
  errno = n;
  err_sys("pthread_rwlock_tryrdlock error");
 }
 return(n);
}
{% endhighlight %}

## 读写锁的写入锁

**pthread_rwlock_wrlock** 只要有读出者持有由调用者指定的读写锁的读出锁，或者有一个写入者持有该读写锁的唯一写入锁（两者都是 rw_refcount 不为 0 的情况），调用线程就得阻塞。为此，我们把 rw_nwaitwriters 加 1，然后在 rw_condwriters 条件变量上调用 pthread_cond_wait。我们将看到，向该条件变量发送信号的前提是：它所在的读写锁被释放，并且有写入者正在等待。取得写入锁后把 rw_refcount 置为 -1。

{% highlight c %}
/* include wrlock */
#include "unpipc.h"
#include "pthread_rwlock.h"

int pthread_rwlock_wrlock(pthread_rwlock_t *rw)
{
 int result;

 if (rw->rw_magic != RW_MAGIC)
  return(EINVAL);

 if ((result = pthread_mutex_lock(&rw->rw_mutex)) != 0)
  return(result);

 while (rw->rw_refcount != 0) {
  rw->rw_nwaitwriters++;
  result = pthread_cond_wait(&rw->rw_condwriters, &rw->rw_mutex);
  rw->rw_nwaitwriters--;
	if (result != 0)
   break;
 }
 if (result == 0)
  rw->rw_refcount = -1;

 pthread_mutex_unlock(&rw->rw_mutex);
 return(result);
}
/* end wrlock */

void Pthread_rwlock_wrlock(pthread_rwlock_t *rw)
{
 int	n;

 if ((n = pthread_rwlock_wrlock(rw)) == 0)
  return;
 errno = n;
 err_sys("pthread_rwlock_wrlock error");
}
{% endhighlight %}

## 读写锁的非阻塞写入锁

非阻塞 **pthread_rwlock_trywrlock** 函数，如果 rw_refcount 不为 0，那么由调用者指定的读写锁或者由一个写入者持有，或者由一个或多个读出者持有（至于由哪个持有则无关紧要），因而返回一个 EBUSY 错误。否则，获取该读写锁的写入锁，并把 rw_refcount 置为 -1。

{% highlight c %}
/* include trywrlock */
#include "unpipc.h"
#include "pthread_rwlock.h"

int pthread_rwlock_trywrlock(pthread_rwlock_t *rw)
{
 int result;

 if (rw->rw_magic != RW_MAGIC)
  return(EINVAL);

 if ((result = pthread_mutex_lock(&rw->rw_mutex)) != 0)
  return(result);

 if (rw->rw_refcount != 0)
  result = EBUSY; /* held by either writer or reader(s) */
 else
  rw->rw_refcount = -1;	/* available, indicate a writer has it */

 pthread_mutex_unlock(&rw->rw_mutex);
 return(result);
}
/* end trywrlock */

int Pthread_rwlock_trywrlock(pthread_rwlock_t *rw)
{
 int n;

 if ((n = pthread_rwlock_trywrlock(rw)) != 0) {
  errno = n;
  err_sys("pthread_rwlock_trywrlock error");
 }
 return(n);
}
{% endhighlight %}

## 读写锁的释放锁

**pthread_rwlock_unlock** 的实现，如果 rw_refcount 当前大于 0，那么有一个读出者（即调用线程）准备释放一个读出锁。如果 rw_refcount 当前为 -1，那么有一个写入者（即调用线程）准备释放一个写入锁。如果有一个写入者在等待，那么一旦由调用者指定的读写锁变得可用（也就是说它的引用计数变为 0），就向 rw_condwriters 条件变量发送信号。我们知道只有一个写入者能够获取该读写锁，因此调用 pthread_cond_signal 来唤醒一个线程。如没有写入者在等待，但是有一个或多个读出者在等待，那就在 rw_condreaders 条件变量上调用 pthread_cond_broadcast，因为所有等待着的读出者都可以获取一个读出锁。注意，一旦有一个写入者在等待，我们就不给任何读出者授予读出锁，否则一个持续的读请求流可能永远阻塞某个等待着的写入者。由于这个原因，我们需要两个分开的 if 条件测试。

{% highlight c %}
/* include unlock */
#include "unpipc.h"
#include "pthread_rwlock.h"

int pthread_rwlock_unlock(pthread_rwlock_t *rw)
{
 int result;

 if (rw->rw_magic != RW_MAGIC)
  return(EINVAL);

 if ((result = pthread_mutex_lock(&rw->rw_mutex)) != 0)
  return(result);

 if (rw->rw_refcount > 0)
  rw->rw_refcount--; /* releasing a reader */
 else if (rw->rw_refcount == -1)
  rw->rw_refcount = 0; /* releasing a reader */
 else
  err_dump("rw_refcount = %d", rw->rw_refcount);

 /* 4give preference to waiting writers over waiting readers */
 if (rw->rw_nwaitwriters > 0) {
  if (rw->rw_refcount == 0)
   result = pthread_cond_signal(&rw->rw_condwriters);
 } else if (rw->rw_nwaitreaders > 0)
  result = pthread_cond_broadcast(&rw->rw_condreaders);

 pthread_mutex_unlock(&rw->rw_mutex);
 return(result);
}
/* end unlock */

void Pthread_rwlock_unlock(pthread_rwlock_t *rw)
{
 int n;

 if ((n = pthread_rwlock_unlock(rw)) == 0)
  return;
 errno = n;
 err_sys("pthread_rwlock_unlock error");
}
{% endhighlight %}

## 参考资料

* [UNIX 网络编程 - 进程间通信 - 第三部分 同步](https://book.douban.com/subject/4118577/)
