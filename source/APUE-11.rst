线程
----

线程概念
~~~~~~~~

每个线程都包含有表示执行环境所必需的信息，其中包括：进程中标识线程的线程ID、一组寄存器值、栈、调度优先级和策略、信号屏蔽字、errno变量以及线程私有数据。

一个进程的所有信息对该进程的所有线程都是共享的，包括可执行程序的代码、程序的全局内存和堆内存、栈以及文件描述符。

线程标识
~~~~~~~~

每个进程有一个进程ID一样，每个线程也有一个线程ID。进程ID在整个系统中是唯一的，但线程ID不同，线程ID只有在它所属的进程上下文中才有意义。

用\ ``pthread_t``\ 类型来表示线程ID，为了可移植性，需要使用pthread_equal函数来比较两个线程ID：

.. code:: c

   #include <pthread.h>

   int pthread_equal(pthread_t tid1, pthread_t tid2);
   // 相等返回非0数值，不相等返回0

可以调用pthread_self函数来获取自己的线程ID：

.. code:: c

   #include <pthread.h>

   pthread_t pthread_self(void);
   // 返回调用线程的线程ID

线程创建
~~~~~~~~

可以通过pthread_create来创建一个新线程：

.. code:: c

   #include <pthread.h>

   int pthread_create(pthread_t* restrict tidp, const pthread_attr_t* restrict attr, void* (*start_rtn)(void*), void* restrict arg);
   // 成功返回0，出错返回错误编号

当成功返回时，tidp指向的内存区域存放着新线程的ID。

attr参数用于定制各种不同的线程属性，可以传入NULL来设置为默认属性。

新创建的线程从start_rtn函数的地址开始运行，该函数只有一个无类型指针参数arg。如果需要向start_rtn函数传递的参数有一个以上，那么需要把这些参数放到一个结构中，然后把这个结构的地址作为arg参数传入。

线程终止
~~~~~~~~

如果进程中的任意线程调用了exit、_Exit或者_exit，那么整个进程就会终止。

与此相类似，如果默认的动作是终止进程，那么，发送到线程的信号就会终止整个进程。

线程可以通过3种方式退出，并且可以在不终止整个进程的情况下，停止它的控制流：

1. 简单地从启动例程中返回，返回值是线程的退出码。

2. 被同一进程中的其他线程取消。

3. 调用pthread_exit。

.. code:: c

   #include <pthread.h>

   void pthread_exit(void* rval_ptr);

rval_ptr的值会作为线程的返回值（注意，不是rval_ptr指向的数据而是rval_ptr本身）。

可以使用pthread_join来获取线程的返回值：

.. code:: c

   #include <pthread.h>

   int pthread_join(pthread_t thread, void** rval_ptr);
   // 成功返回0，出错返回错误编号

调用线程将一直阻塞，直到指定的线程调用pthread_exit、从启动例程中返回或者被取消。

-  如果线程简单地从它的启动例程返回，rval_ptr就指向返回值。
-  如果线程被取消，由rval_ptr指定的内存单元就设置为PTHREAD_CANCELED。

pthread_join自动把线程置于分离状态。如果线程已经处于分离状态，pthread_join调用就会失败，返回EINVAL。

如果对线程的返回值并不感兴趣，那么可以把rval_ptr设置为NULL。

一个简单的示例：

.. code:: c

   #include "apue.h"
   #include <pthread.h>

   void* thr_fn1(void* arg) {
       printf("thread 1 returning\n");
       return ((void*)1);
   }

   void* thr_fn2(void* arg) {
       printf("thread 2 returning\n");
       pthread_exit((void*)2);
   }

   int main() {
       int err;
       pthread_t tid1, tid2;
       void* tret;
       err = pthread_create(&tid1, NULL, thr_fn1, NULL);
       if (err != 0) {
           err_exit(err, "can't create thread 1");
       }
       err = pthread_create(&tid2, NULL, thr_fn2, NULL);
       if (err != 0) {
           err_exit(err, "can't create thread 2");
       }
       err = pthread_join(tid1, &tret);
       if (err != 0) {
           err_exit(err, "can't join with thread 1");
       }
       printf("thread 1 exit code %ld\n", (long)tret);
       err = pthread_join(tid2, &tret);
       if (err != 0) {
           err_exit(err, "can't join with thread 2");
       }
       printf("thread 2 exit code %ld\n", (long)tret);
       exit(0);
   }

线程可以通过调用pthread_cancel函数来请求取消同一进程中的其他线程。

.. code:: c

   #include <pthread.h>

   int pthread_cancel(pthread_t tid);
   // 成功返回0，出错返回错误编号

在默认情况下，pthread_cancel
函数会使得由tid标识的线程的行为表现为如同调用thread_exit(PTHREAD_CANCELED)。线程可以选择忽略cancel或者控制如何被cancel。

线程可以通过pthread_cleanup_push和pthread_cleanup_pop来注册\ **线程清理处理程序**\ （类似atexit函数，可以建立多个，并且调用顺序与注册顺序相反）：

.. code:: c

   #include <pthread.h>

   void pthread_cleanup_push(void (*rtn)(void*), void* arg);
   void pthread_cleanup_pop(int execute);

只有以下几种情况会触发这些线程清理处理程序：

-  调用pthread_exit时；

-  响应取消请求时；

-  用非零execute参数调用pthread_cleanup_pop时。

如果execute参数设置为0，清理函数将不被调用。

   pthread_cleanup_pop(0)用来和pthread_cleanup_push配套。

   在linux里，这两个函数是用宏来实现的。不配套的话编译就无法通过。

一个使用示例：

.. code:: c

   #include "apue.h"
   #include <pthread.h>

   void cleanup(void *arg) {
       printf("cleanup: %s\n", (char *) arg);
   }

   void *thr_fn1(void *arg) {
       printf("thread 1 start\n");
       pthread_cleanup_push(cleanup, "thread 1 first handler") ;
               pthread_cleanup_push(cleanup, "thread 1 second handler") ;
                       printf("thread 1 push complete\n");
                       if (arg)
                           return ((void *) 1);
               pthread_cleanup_pop(0);
       pthread_cleanup_pop(0);
       return ((void *) 1);
   }
   void *thr_fn2(void *arg) {
       printf("thread 2 start\n");
       pthread_cleanup_push(cleanup, "thread 2 first handler") ;
               pthread_cleanup_push(cleanup, "thread 2 second handler") ;
                       printf("thread 2 push complete\n");
                       if (arg)
                            pthread_exit((void *) 2);
               pthread_cleanup_pop(0);
       pthread_cleanup_pop(0);
       pthread_exit((void *) 2);
   }

   int main() {
       int err;
       pthread_t tid1, tid2;
       void *tret;
       err = pthread_create(&tid1, NULL, thr_fn1, (void*)1);
       if (err != 0) {
           err_exit(err, "can't create thread 1");
       }
       err = pthread_create(&tid2, NULL, thr_fn2, (void*)1);
       if (err != 0) {
           err_exit(err, "can't create thread 2");
       }
       err = pthread_join(tid1, &tret);
       if (err != 0) {
           err_exit(err, "can't join with thread 1");
       }
       printf("thread 1 exit code %ld\n", (long)tret);
       err = pthread_join(tid2, &tret);
       if (err != 0) {
           err_exit(err, "can't join with thread 2");
       }
       printf("thread 2 exit code %ld\n", (long)tret);
       exit(0);
   }

输出结果(每次运行的结果可能不同)：

::

   thread 1 start
   thread 1 push complete
   thread 2 start
   thread 2 push complete
   thread 1 exit code 1
   cleanup: thread 2 second handler
   cleanup: thread 2 first handler
   thread 2 exit code 2

线程同步
~~~~~~~~

互斥量
^^^^^^

互斥变量用pthread_mutex_t类型表示。

使用互斥变量前，需要对其进行初始化，可以把它设置为常量PTHREAD_MUTEX_INITIALIZER或者调用pthread_mutex_init函数来进行初始化。

如果动态分配互斥量，那么释放内存前需要调用pthread_mutex_destroy。

调用pthread_mutex_lock来对互斥量进行加锁。如果互斥量已经上锁，调用线程将阻塞直到互斥量被解锁。调用pthread_mutex_unlock对互斥量解锁。

如果线程不希望被阻塞，可以使用pthread_mutex_trylock尝试对互斥量进行加锁。

-  如果调用pthread_mutex_trylock时互斥量处于未锁住状态，那么pthread_mutex_trylock将锁住互斥量，不会出现阻塞直接返回0，
-  否则pthread_mutex_trylock就会失败，不能锁住互斥量，返回EBUSY。

.. code:: c

   #include <pthread.h>

   int pthread_mutex_init(pthread_mutex_t* restrict mutex, const pthread_mutexattr_t* restrict attr);  // attr为NULL时使用默认配置初始化
   int pthread_mutex_destroy(pthread_mutex_t* mutex);
   int pthread_mutex_lock(pthread_mutex_t* mutex);
   int pthread_mutex_unlock(pthread_mutex_t* mutex);
   int pthread_mutex_trylock(pthread_mutex_t* mutex);
   // 成功返回0，出错返回错误编号

当线程试图获取一个已加锁的互斥量时，pthread_mutex_timedlock互斥量原语允许绑定线程阻塞时间。pthread_mutex_timedlock函数与pthread_mutex_lock是基本等价的，但是在达到超时时间值时，pthread_mutex_timedlock不会对互斥量进行加锁，而是返回错误码ETIMEDOUT。

.. code:: c

   #include <pthread.h>
   #include <time.h>

   int pthread_mutex_timedlock(pthread_mutex_t* restrict mutex, const struct timespec* restrict tsptr);
   // 成功返回0，出错返回错误编号

一个使用示例：

.. code:: c

   #include "apue.h"
   #include <pthread.h>

   int main() {
       int err;
       struct timespec tout;
       struct tm* tmp;
       char buf[64];
       pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

       pthread_mutex_lock(&lock);
       printf("mutex is locked\n");
       clock_gettime(CLOCK_REALTIME, &tout);
       tmp = localtime(&tout.tv_sec);
       strftime(buf, sizeof(buf), "%r", tmp);
       printf("current time is %s\n", buf);
       tout.tv_sec += 10;
       /* this could lead to deadlock */
       err = pthread_mutex_timedlock(&lock, &tout);
       clock_gettime(CLOCK_REALTIME, &tout);
       tmp = localtime(&tout.tv_sec);
       strftime(buf, sizeof(buf), "%r", tmp);
       printf("current time is %s\n", buf);
       if (err == 0) {
           printf("mutex locked again\n");
       } else {
           printf("can't lock mutex again: %s\n", strerror(err));
       }
       exit(0);
   }

读写锁
^^^^^^

读写锁可以有3种状态：读模式下加锁状态，写模式下加锁状态，不加锁状态。一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁。

当读写锁是写加锁状态时，在这个锁被解锁之前，所有试图对这个锁加锁的线程都会被阻塞。当读写锁在读加锁状态时，所有试图以读模式对它进行加锁的线程都可以得到访问权，但是任何希望以写模式对此锁进行加锁的线程都会阻塞，直到所有的线程释放它们的读锁为止。当读写锁处于读模式锁住的状态，而这时有一个线程试图以写模式获取锁时，读写锁通常会阻塞随后的读模式锁请求。

简而言之，这是一个\ **写者优先**\ 的读写锁。

   读写锁也叫做共享互斥锁（shared-exclusive
   lock）。当读写锁是读模式锁住时，就可以说成是以共享模式锁住的。当它是写模式锁住的时候，就可以说成是以互斥模式锁住的。

与互斥量相比，读写锁在使用之前必须初始化(使用PTHREAD_RWLOCK_INITIALIZER或者调用pthread_rwlock_init函数)，在释放它们底层的内存之前必须销毁(调用pthread_rwlock_destroy函数)。

要在读模式下锁定读写锁，需要调用pthread_rwlock_rdlock。要在写模式下锁定读写锁，需要调用pthread_rwlock_wrlock。不管以何种方式锁住读写锁，都可以调用pthread_rwlock_unlock进行解锁。

SUS还定义了读写锁原语的条件版本(trylock版本)。

.. code:: c

   #include <pthread.h>

   int pthread_rwlock_init(pthread_rwlock_t* restrict rwlock, const pthread_rwlockattr_t* restrict attr);
   int pthread_rwlock_destroy(pthread_rwlock_t* rwlock);
   int pthread_rwlock_rdlock(pthread_rwlock_t* rwlock);
   int pthread_rwlock_wrlock(pthread_rwlock_t* rwlock);
   int pthread_rwlock_unlock(pthread_rwlock_t* rwlock);
   int pthread_rwlock_tryrdlock(pthread_rwlock_t* rwlock);
   int pthread_rwlock_trywrlock(pthread_rwlock_t* rwlock);
   // 成功返回0，出错返回错误编号

一个使用示例：

.. code:: c

   #include <stdlib.h>
   #include <pthread.h>

   struct job {
       struct job* j_next;
       struct job* j_prev;
       pthread_t j_id;
       // more stuff here
   };

   struct queue {
       struct job* q_head;
       struct job* q_tail;
       pthread_rwlock_t q_lock;
   };

   int queue_init(struct queue* qp) {
       int err;
       qp->q_head = NULL;
       qp->q_tail = NULL;
       err = pthread_rwlock_init(&qp->q_lock, NULL);
       if (err != 0) {
           return err;
       }
       return 0;
   }

   void job_insert(struct queue* qp, struct job* jp) {
       pthread_rwlock_wrlock(&qp->q_lock);
       jp->j_next = qp->q_head;
       jp->j_prev = NULL;
       if (qp->q_head != NULL) {
           qp->q_head->j_prev = jp;
       } else {
           qp->q_tail = jp;
       }
       qp->q_head = jp;
       pthread_rwlock_unlock(&qp->q_lock);
   }

   void job_append(struct queue* qp, struct job* jp) {
       pthread_rwlock_wrlock(&qp->q_lock);
       jp->j_next = NULL;
       jp->j_prev = qp->q_tail;
       if (qp->q_tail != NULL) {
           qp->q_tail->j_next = jp;
       } else {
           qp->q_head = jp;
       }
       qp->q_tail = jp;
       pthread_rwlock_unlock(&qp->q_lock);
   }

   void job_remove(struct queue* qp, struct job* jp) {
       pthread_rwlock_wrlock(&qp->q_lock);
       if (jp == qp->q_head) {
           qp->q_head = jp->j_next;
           if (qp->q_tail == jp) {
               qp->q_tail = NULL;
           } else {
               jp->j_next->j_prev = jp->j_prev;
           }
       } else if (jp == qp->q_tail) {
           qp->q_tail = jp->j_prev;
           jp->j_prev->j_next = jp->j_next;
       } else {
           jp->j_prev->j_next = jp->j_next;
           jp->j_next->j_prev = jp->j_prev;
       }
       pthread_rwlock_unlock(&qp->q_lock);
   }

   struct job* job_find(struct queue* qp, pthread_t id) {
       struct job* jp;
       if (pthread_rwlock_rdlock(&qp->q_lock) != 0) {
           return NULL;
       }
       for (jp = qp->q_head; jp != NULL; jp = jp->j_next) {
           if (pthread_equal(jp->j_id, id)) {
               break;
           }
       }
       pthread_rwlock_unlock(&qp->q_lock);
       return jp;
   }

SUS同样提供了读写锁的带有超时的加锁函数：

.. code:: c

   #include <pthread.h>
   #include <time.h>

   int pthread_rwlock_timedrdlock(pthread_rwlock_t* restrict rwlock, const struct timespec* restrict tsptr);
   int pthread_rwlock_timedwlock(pthread_rwlock_t* restrict rwlock, const struct timespec* restrict tsptr);
   // 成功返回0，出错返回错误编号

条件变量
~~~~~~~~

条件变量的初始化和摧毁与之前的互斥量和读写锁差不多：

.. code:: c

   #include <pthread.h>

   int pthread_cond_init(pthread_cond_t* restrict cond, const pthread_condattr_t* restrict attr);
   int pthread_cond_destroy(pthread_cond_t* cond);
   // 成功返回0，出错返回错误编号

使用pthread_cond_wait来等待条件变量为真：

.. code:: c

   #include <pthread.h>

   int pthread_cond_wait(pthread_cond_t* restrict cond, pthread_mutex_t* restrict mutex);
   int pthread_cond_timedwait(pthread_cond_t* restrict cond, pthread_mutex_t* restrict mutex, const struct timespec* restrict tsptr);
   // 成功返回0，出错返回-1

传递给pthread_cond_wait的互斥量对条件进行保护。调用者把锁住的互斥量传给函数，函数然后自动把调用线程放到等待条件的线程列表上，对互斥量解锁。这就关闭了条件检查和线程进入休眠状态等待条件改变这两个操作之间的时间通道，这样线程就不会错过条件的任何变化。pthread_cond_wait返回时，互斥量再次被锁住。

有两个函数可以用于通知线程条件已经满足。

.. code:: c

   #include <pthread.h>

   int pthread_cond_signal(pthread_cond_t *cond);
   int pthread_cond_broadcast(pthread_cond_t *cond);
   // 成功返回0，出错返回错误编号

pthread_cond_signal函数至少能唤醒一个等待该条件的线程，而pthread_cond_broadcast函数则能唤醒等待该条件的所有线程。

   POSIX规范为了简化pthread_cond_signal的实现，允许它在实现的时候唤醒一个以上的线程。

使用示例：

.. code:: c

   #include <pthread.h>

   struct msg {
       struct msg* m_next;
       // more stuff here
   };

   struct msg* workq;
   pthread_cond_t qready = PTHREAD_COND_INITIALIZER;
   pthread_mutex_t qlock = PTHREAD_MUTEX_INITIALIZER;

   void process_msg() {
       struct msg* mp;
       for( ; ; ) {
           pthread_mutex_lock(&qlock);
           while (workq == NULL) {
               pthread_cond_wait(&qready, &qlock);
           }
           mp = workq;
           workq = mp->m_next;
           pthread_mutex_unlock(&qlock);
           /* now process the message up */
       }
   }

   void enqueue_msg(struct msg* mp) {
       pthread_mutex_lock(&qlock);
       mp->m_next = workq;
       workq = mp;
       pthread_mutex_unlock(&qlock);
       pthread_cond_signal(&qready);
   }

自旋锁
^^^^^^

.. code:: c

   #include <pthread.h>

   int pthread_spin_init(pthread_spinlock_t* lock, int pshared);
   int pthread_spin_destroy(pthread_spinlock_t* lock);
   int pthread_spin_lock(pthread_spinlock_t* lock);
   int pthread_spin_unlock(pthread_spinlock_t* lock);
   int pthread_spin_trylock(pthread_spinlock_t* lock);
   // 成功返回0，出错返回错误编号

pshared参数表示进程共享属性：

-  如果为PTHREAD_PROCESS_SHARED，则自旋锁能被可以访问锁底层内存的线程所获取，即便那些线程属于不同的进程，情况也是如此。
-  如果为PTHREAD_PROCESS_PRIVATE，自旋锁就只能被初始化该锁的进程内部的线程所访问。

如果自旋锁当前在解锁状态的话，pthread_spin_lock函数不要自旋就可以对它加锁。如果线程已经对它加锁了，结果就是未定义的。调用pthread_spin_lock会返回EDEADLK错误（或其他错误），或者调用可能会永久自旋。具体行为依赖于实际的实现。试图对没有加锁的自旋锁进行解锁，结果也是未定义的。

屏障
^^^^

屏障（barrier）是用户协调多个线程并行工作的同步机制。屏障允许每个线程等待，直到所有的合作线程都到达某一点，然后从该点继续执行。

.. code:: c

   #include <pthread.h>

   int pthread_barrier_init(pthread_barrier_t *restrict barrier, const pthread_barrierattr_t *restrict attr, unsigned int count);
   int pthread_barrier_destroy(pthread_barrier_t *barrier);
   // 若成功返回0，出错返回错误编号

count参数指定在允许所有线程继续运行之前，必须到达屏障的线程数目。

attr参数指定屏障对象的属性（NULL用默认属性初始化屏障）。

可以使用pthread_barrier_wait函数来表明，线程已完成工作，准备等所有其他线程赶上来。

.. code:: c

   #include <pthread.h>

   int pthread_barrier_wait(pthread_barrier_t *barrier);
   // 成功返回0或者PTHREAD_BARRIER_SERIAL_THREAD，出错返回错误编号

调用pthread_barrier_wait的线程在屏障计数（调用pthread_barrier_init时设定）未满足条件时，会进入休眠状态。如果该线程是最后一个调用pthread_barrier_wait的线程，就满足了屏障计数，所有的线程都被唤醒。

   对于一个任意线程，pthread_barrier_wait函数返回了PTHREAD_BARRIER_SERIAL_THREAD。剩下的线程看到的返回值是0。这使得一个线程可以作为主线程，它可以工作在其他所有线程已完成的工作结果上。

一旦达到屏障计数值，而且线程处于非阻塞状态，屏障就可以被重用。但是除非在调用了pthread_barrier_destroy函数之后，又调用了pthread_barrier_init函数对计数用另外的数进行初始化，否则屏障计数不会改变。

使用示例：

.. code:: c

   #include "apue.h"
   #include <pthread.h>
   #include <limits.h>
   #include <sys/time.h>

   #define NTHR 8              // num of threads
   #define NUMNUM 8000000L     // num of numbers to sort
   #define TNUM (NUMNUM/NTHR)  // num per thread

   long nums[NUMNUM];
   long snums[NUMNUM];

   pthread_barrier_t b;

   #ifdef SOLARIS
   #define heapsort qsort
   #else
   extern int heapsort(void*, size_t, size_t, int (*)(const void*, const void*));
   #endif

   int complong(const void* arg1, const void* arg2) {
       long l1 = *(long*)arg1;
       long l2 = *(long*)arg2;
       if (l1 == l2) {
           return 0;
       } else if (l1 < l2) {
           return -1;
       } else {
           return 1;
       }
   }

   void* thr_fn(void* arg) {
       long idx = (long)arg;
       heapsort(&nums[idx], TNUM, sizeof(long), complong);
       pthread_barrier_wait(&b);
       return ((void*)0);
   }

   void merge() {
       long idx[NTHR];
       long i, minidx, sidx, num;
       for (i = 0; i < NTHR; i++) {
           idx[i] = i * TNUM;
       }
       for (sidx = 0; sidx < NUMNUM; sidx++) {
           num = LONG_MAX;
           for (i = 0; i < NTHR; i++) {
               if ((idx[i] < (i + 1) * TNUM) && (nums[idx[i]] < num)) {
                   num = nums[idx[i]];
                   minidx = i;
               }
           }
           snums[sidx] = nums[idx[minidx]];
           idx[minidx]++;
       }
   }

   int main() {
       unsigned long i;
       struct timeval start, end;
       long long startusec, endusec;
       double elapsed;
       int err;
       pthread_t tid;
       srandom(1);
       for (i = 0; i < NUMNUM; i++) {
           nums[i] = random();
       }
       gettimeofday(&start, NULL);
       pthread_barrier_init(&b, NULL, NTHR + 1);
       for (i = 0; i < NTHR; i++) {
           err = pthread_create(&tid, NULL, thr_fn, (void*)(i * TNUM));
           if (err != 0) {
               err_exit(err, "can't create thread");
           }
       }
       pthread_barrier_wait(&b);
       merge();
       gettimeofday(&end, NULL);
       startusec = start.tv_sec * 1000000 + start.tv_usec;
       endusec = end.tv_sec * 1000000 + end.tv_usec;
       elapsed = (double)(endusec - startusec) / 1000000.0;
       printf("sort took %.4f seconds\n", elapsed);
       for (i = 0; i < NUMNUM; i++) {
           printf("%ld\n", snums[i]);
       }
       exit(0);
   }

..

   此代码在linux上无法运行，找不到heapsort函数。（可能需要下载bsd的头文件）
