---
title: "Timer定时器的设计和实现"
date: 2017-04-08T21:07:16+08:00
draft: false
---
#### 定时器
在一般的服务端程序设计中，与时间有关的常见任务有：

1. 获取当前时间，计算时间间隔
2. 时区转换与日期计算；把纽约当地时间转换成上海当地时间；2011-02-05之后的100天是几月几号星期几;等等。
3. 定时操作，比如在预定的时间执行任务，或者在一段延时之后执行任务。

这里我们讨论第3项。Linux计时函数有下面这些：

- time(2) / time_t (s)
- ftime(3) / struct timeb (ms)
- gettimeofday(2) / struct timeval (us)
- clock_gettime(2) / struct timespec (ns)

定时函数，用于让程序等待一段时间或安排计划任务：

- sleep(3)
- alarm(2)
- usleep(3)
- nanosleep(2)
- clock_nanosleep(2)
- gettimer(2) / settimer(2)
- timer_create(2) / timer_settime(2) / timer_gettime(2) / timer_delete(2)
- timerfd_create(2) / timerfd_gettime(2) / timerfd_settime(2)

我的取舍如下：

- (计时) 只使用gettimeofday(2)来获取当前时间
- (定时) 只使用timerfd_*系列函数.

因为，`gettimeofday(2)`的精度足够，`timerfd_create(2)` 把时间变成一个文件描述符，该“文件”在定时器超时那一刻变得可读，这样就能很方便地融入`select(2)/poll(2)`框架中，用统一的方式来处理IO事件和超时事件。

#### 定时器数据结构
TimerQueue需要高效地组织目前尚未到期的Timer，能够快速地根据当前时间找到已经到期的Timer，也要能高效添加和删除Timer。最简单的TimerQueue以按照到期时间排好序的线性表为数据结构，查找复杂度为O(N)。

另外一种做法是用大顶堆或小顶堆，这样复杂度降为O(logN)，但是C++标准库的make_heap()等函数不能高效地删除heap中间的某个元素，需要我们自己实现。

还有一种做法是使用二叉搜索树(如std::set/std::map)，把Timer按到期时间先后排好序。操作的复杂度仍然是O(logN)，不过memeory locality比heap要差一些,实际速度可能略慢。 但是我们不能够直接用map<Timestamp, Timer*>，因为这样无法处理两个Timer到期时间相同的情况。可以用std::pair<Timestamp, Timer*>为key, 这样即便两个Timer到期时间相同，他们的地址也是不同的。下面是TimerQueue的接口:

```cpp
class TimerQueue : boost::noncopyable
{
public:
    TimerQueue(EventLoop* loop);
    ~TimerQueue();

    ///
    /// Schedules the callback to be run at given time,
    /// repeats if @c interval > 0.0.
    ///
    /// Must be thread safe. Usually be called from other threads.
    TimerId addTimer(const TimerCallback& cb,
            Timestamp when,
            double interval);
#ifdef __GXX_EXPERIMENTAL_CXX0X__
    TimerId addTimer(TimerCallback&& cb,
            Timestamp when,
            double interval);
#endif

    void cancel(TimerId timerId);

private:

    // FIXME: use unique_ptr<Timer> instead of raw pointers.
    typedef std::pair<Timestamp, Timer*> Entry;
    typedef std::set<Entry> TimerList;
    typedef std::pair<Timer*, int64_t> ActiveTimer;
    typedef std::set<ActiveTimer> ActiveTimerSet;

    void addTimerInLoop(Timer* timer);
    void cancelInLoop(TimerId timerId);
    // called when timerfd alarms
    void handleRead();
    // move out all expired timers
    std::vector<Entry> getExpired(Timestamp now);
    void reset(const std::vector<Entry>& expired, Timestamp now);

    bool insert(Timer* timer);

    EventLoop* loop_;
    const int timerfd_;
    Channel timerfdChannel_;
    // Timer list sorted by expiration
    TimerList timers_;

    // for cancel()
    ActiveTimerSet activeTimers_;
    bool callingExpiredTimers_; /* atomic */
    ActiveTimerSet cancelingTimers_;
};
```

其中getExpired的实现如下：

```cpp
std::vector<TimerQueue::Entry> TimerQueue::getExpired(Timestamp now)
{
    assert(timers_.size() == activeTimers_.size());
    std::vector<Entry> expired;
    Entry sentry(now, reinterpret_cast<Timer*>(UINTPTR_MAX));
    TimerList::iterator end = _timers.lower_bound(sentry);
    assert(end == _timers.end() || now < end->first);
    std::copy(_timers.begin(), end, back_inserter(expired));
    _timers.erase(_timers.begin(), end);

    for (std::vector<Entry>::iterator it = expired.begin();
            it != expired.end(); ++it) {
        ActiveTimer timer(it->second, it->second->sequence());
        size_t n = _activeTimers.erase(timer);
        assert(n == 1); (void)n;
    }

    assert(_timers.size() == _activeTimers.size());
    return expired;
}

```
