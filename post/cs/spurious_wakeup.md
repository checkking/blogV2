---
title: "pthread_cond_wait的虚假唤醒"
date: 2017-03-06T21:07:16+08:00
draft: false
---
#### pthread_cond_wait通常用法
pthread_cond_wait的通常使用方法如下：
```c
#include <pthread.h>
struct msg {
    struct msg *m_next;
    /* ... more stuff here ... */
};
struct msg *workq;
pthread_cond_t qready = PTHREAD_COND_INITIALIZER;
pthread_mutex_t qlock = PTHREAD_MUTEX_INITIALIZER;
void process_msg(void) {
    struct msg *mp;
    for (;;) {
        pthread_mutex_lock(&qlock);
        while (workq == NULL) {       // (1)
            pthread_cond_wait(&qready, &qlock);
        }
    }
    mp = workq;
    workq = mp->m_next;
    pthread_mutex_unlock(&qlock);
    /* now process the message mp */
}

void enqueue_msg(struct msg *mp) {
    pthread_mutex_lock(&qlock);
    mp->m_next = workq;
    workq = mp;
    pthread_mutex_unlock(&qlock);
    pthread_cond_signal(&qready);
}
```
(1)处为什么要用while, 而不是简单的if呢？这是因为为了避免Spurious wakeup。
#### Spurious Wakeup
spurious wakeup 指的是一次 signal() 调用唤醒两个或以上 wait()ing 的线程，或者没有调用 signal() 却有线程从 wait() 返回。
在linux中，pthread_cond_wait底层是futex系统调用。在linux中，任何慢速的阻塞的系统调用当接收到信号的时候，就会返回-1，并且设置errno为EINTR。在系统调用返回前，用户程序注册的信号处理函数会被调用处理。
实际上，不仅仅信号会导致假唤醒，pthread_cond_broadcast也会导致假唤醒。加入条件变量上有多个线程在等待，pthread_cond_broadcast会唤醒所有的等待线程，而pthread_cond_signal只会唤醒其中一个等待线程。这样，pthread_cond_broadcast的情况也许要在pthread_cond_wait前使用while循环来检查条件变量。
