在开始介绍播放器开发之前，我们首先对posix库进行一定的封装，得到我们想要的Mutex、Condition、Thread等类。至于为何不用C++11自带的相关类呢？这是考虑到编译环境的问题，有些公司可能仍旧没升级NDK的版本，不支持C++11，这里为了方便，只好利用Posix封装一套Thread相关的基础类，部分代码参考(copy)自Android 源码中的代码。至于原理，这里就不介绍了，网上相关资料还是很多的，分析互斥锁、条件锁等原理不是本文章的重点。

### Mutex封装
Mutex的封装可参考Android 的libutil库里面的代码，直接复制过来使用即可，代码里面还封装了AutoLock。代码如下：
```
#ifndef MUTEX_H
#define MUTEX_H

#include <stdint.h>
#include <sys/types.h>
#include <time.h>

#include <pthread.h>

typedef int32_t     status_t;

class Condition;

class Mutex {
public:
    enum {
        PRIVATE = 0,
        SHARED = 1
    };
    Mutex();
    Mutex(const char* name);
    Mutex(int type, const char* name = NULL);
    ~Mutex();

    // lock or unlock the mutex
    status_t    lock();
    void        unlock();

    // lock if possible; returns 0 on success, error otherwise
    status_t    tryLock();

    // Manages the mutex automatically. It'll be locked when Autolock is
    // constructed and released when Autolock goes out of scope.
    class Autolock {
    public:
        inline Autolock(Mutex& mutex) : mLock(mutex)  { mLock.lock(); }
        inline Autolock(Mutex* mutex) : mLock(*mutex) { mLock.lock(); }
        inline ~Autolock() { mLock.unlock(); }
    private:
        Mutex& mLock;
    };

private:
    friend class Condition;

    // A mutex cannot be copied
    Mutex(const Mutex&);
    Mutex&      operator = (const Mutex&);

    pthread_mutex_t mMutex;
};

inline Mutex::Mutex() {
    pthread_mutex_init(&mMutex, NULL);
}

inline Mutex::Mutex(const char* name) {
    pthread_mutex_init(&mMutex, NULL);
}


inline Mutex::Mutex(int type, const char* name) {
    if (type == SHARED) {
        pthread_mutexattr_t attr;
        pthread_mutexattr_init(&attr);
        pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);
        pthread_mutex_init(&mMutex, &attr);
        pthread_mutexattr_destroy(&attr);
    } else {
        pthread_mutex_init(&mMutex, NULL);
    }
}

inline Mutex::~Mutex() {
    pthread_mutex_destroy(&mMutex);
}

inline status_t Mutex::lock() {
    return -pthread_mutex_lock(&mMutex);
}

inline void Mutex::unlock() {
    pthread_mutex_unlock(&mMutex);
}

inline status_t Mutex::tryLock() {
    return -pthread_mutex_trylock(&mMutex);
}
typedef Mutex::Autolock AutoMutex;

#endif //MUTEX_H
```

### Condition封装
Condition类的封装跟Mutex一样，直接从Android源码里面复制过来，稍作修改即可。代码如下：
```
#ifndef CONDITION_H
#define CONDITION_H

#include <stdint.h>
#include <sys/types.h>
#include <time.h>
#include <pthread.h>

#include <Mutex.h>

typedef int64_t nsecs_t; // nano-seconds

class Condition {
public:
    enum {
        PRIVATE = 0,
        SHARED = 1
    };

    enum WakeUpType {
        WAKE_UP_ONE = 0,
        WAKE_UP_ALL = 1
    };

    Condition();
    Condition(int type);
    ~Condition();

    status_t wait(Mutex& mutex);
    status_t waitRelative(Mutex& mutex, nsecs_t reltime);
    void signal();
    void signal(WakeUpType type) {
        if (type == WAKE_UP_ONE) {
            signal();
        } else {
            broadcast();
        }
    }
    void broadcast();
private:
    pthread_cond_t mCond;
};

inline Condition::Condition() {
    pthread_cond_init(&mCond, NULL);
}

inline Condition::Condition(int type) {
    if (type == SHARED) {
        pthread_condattr_t attr;
        pthread_condattr_init(&attr);
        pthread_condattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);
        pthread_cond_init(&mCond, &attr);
        pthread_condattr_destroy(&attr);
    } else {
        pthread_cond_init(&mCond, NULL);
    }
}

inline Condition::~Condition() {
    pthread_cond_destroy(&mCond);
}

inline status_t Condition::wait(Mutex &mutex) {
    return -pthread_cond_wait(&mCond, &mutex.mMutex);
}

inline status_t Condition::waitRelative(Mutex &mutex, nsecs_t reltime) {
    struct timeval t;
    struct timespec ts;
    gettimeofday(&t, NULL);
    ts.tv_sec  = t.tv_sec;
    ts.tv_nsec = t.tv_usec*1000;

    ts.tv_sec  += reltime / 1000000000;
    ts.tv_nsec += reltime % 1000000000;
    if (ts.tv_nsec >= 1000000000) {
        ts.tv_nsec -= 1000000000;
        ts.tv_sec  += 1;
    }
    return -pthread_cond_timedwait(&mCond, &mutex.mMutex, &ts);
}

inline void Condition::signal() {
    pthread_cond_signal(&mCond);
}

inline void Condition::broadcast() {
    pthread_cond_broadcast(&mCond);
}

#endif //CONDITION_H
```

### Thread封装
为了方便使用线程，我们对pthread进行封装。完整的代码如下：
```
#include <Mutex.h>
#include <Condition.h>

typedef enum {
    Priority_Default = -1,
    Priority_Low = 0,
    Priority_Normal = 1,
    Priority_High = 2
} ThreadPriority;

class Runnable {
public:
    virtual ~Runnable(){}

    virtual void run() = 0;
};

/**
 * Thread can use a custom Runnable, but must delete Runnable constructor yourself
 */
class Thread : public Runnable {
public:

    Thread();

    Thread(ThreadPriority priority);

    Thread(Runnable *runnable);

    Thread(Runnable *runnable, ThreadPriority priority);

    virtual ~Thread();

    void start();

    void join();

    void detach();

    pthread_t getId() const;

    bool isActive() const;

protected:
    static void *threadEntry(void *arg);

    int schedPriority(ThreadPriority priority);

    virtual void run();

protected:
    Mutex mMutex;
    Condition mCondition;
    Runnable *mRunnable;
    ThreadPriority mPriority; // thread priority
    pthread_t mId;  // thread id
    bool mRunning;  // thread running
    bool mNeedJoin; // if call detach function, then do not call join function
};

inline Thread::Thread() {
    mNeedJoin = true;
    mRunning = false;
    mId = -1;
    mRunnable = NULL;
    mPriority = Priority_Default;
}

inline Thread::Thread(ThreadPriority priority) {
    mNeedJoin = true;
    mRunning = false;
    mId = -1;
    mRunnable = NULL;
    mPriority = priority;
}

inline Thread::Thread(Runnable *runnable) {
    mNeedJoin = false;
    mRunning = false;
    mId = -1;
    mRunnable = runnable;
    mPriority = Priority_Default;
}

inline Thread::Thread(Runnable *runnable, ThreadPriority priority) {
    mNeedJoin = false;
    mRunning = false;
    mId = -1;
    mRunnable = runnable;
    mPriority = priority;
}

inline Thread::~Thread() {
    join();
    mRunnable = NULL;
}

inline void Thread::start() {
    if (!mRunning) {
        pthread_create(&mId, NULL, threadEntry, this);
        mNeedJoin = true;
    }
    // wait thread to run
    mMutex.lock();
    while (!mRunning) {
        mCondition.wait(mMutex);
    }
    mMutex.unlock();
}

inline void Thread::join() {
    Mutex::Autolock lock(mMutex);
    if (mId > 0 && mNeedJoin) {
        pthread_join(mId, NULL);
        mNeedJoin = false;
        mId = -1;
    }
}

inline  void Thread::detach() {
    Mutex::Autolock lock(mMutex);
    if (mId >= 0) {
        pthread_detach(mId);
        mNeedJoin = false;
    }
}

inline pthread_t Thread::getId() const {
    return mId;
}

inline bool Thread::isActive() const {
    return mRunning;
}

inline void* Thread::threadEntry(void *arg) {
    Thread *thread = (Thread *) arg;

    if (thread != NULL) {
        thread->mMutex.lock();
        thread->mRunning = true;
        thread->mCondition.signal();
        thread->mMutex.unlock();

        thread->schedPriority(thread->mPriority);

        // when runnable is exit，run runnable else run()
        if (thread->mRunnable) {
            thread->mRunnable->run();
        } else {
            thread->run();
        }

        thread->mMutex.lock();
        thread->mRunning = false;
        thread->mCondition.signal();
        thread->mMutex.unlock();
    }

    pthread_exit(NULL);

    return NULL;
}

inline int Thread::schedPriority(ThreadPriority priority) {
    if (priority == Priority_Default) {
        return 0;
    }

    struct sched_param sched;
    int policy;
    pthread_t thread = pthread_self();
    if (pthread_getschedparam(thread, &policy, &sched) < 0) {
        return -1;
    }
    if (priority == Priority_Low) {
        sched.sched_priority = sched_get_priority_min(policy);
    } else if (priority == Priority_High) {
        sched.sched_priority = sched_get_priority_max(policy);
    } else {
        int min_priority = sched_get_priority_min(policy);
        int max_priority = sched_get_priority_max(policy);
        sched.sched_priority = (min_priority + (max_priority - min_priority) / 2);
    }

    if (pthread_setschedparam(thread, policy, &sched) < 0) {
        return -1;
    }
    return 0;
}

inline void Thread::run() {
    // do nothing
}
```
备注：
0. 为何不用C++11的线程？编译器可能不支持C++11。这里只是做兼容，而且音视频的库基本都是C语言编写的，这里主要是考虑到二进制接口兼容性的问题。在使用带异常的C++时，有可能会导致ffmpeg某些版本出现偶然的内部崩溃问题，这个是我在实际使用过程中发现的。这个C++二进制接口兼容性问题各个技术大牛有专门讨论过，我并不擅长C++，也讲不出更深入的说法，想要了解的话，建议自行找资料了解，这里就不费口舌了。

1. 当继承Thread类时，我们需要重写run方法。

2. Runnable 是一个抽象基类，用来模仿Java层的Runnable接口。当我们使用Runnable时，必须有外部释放Runnable的内存，这里并没有垃圾回收功能，要做成Java那样能够自动回收内存，这个超出了我的能力范围。我这里只是为了方便使用而简单地将pthread封装起来使用而已。

3. 如果要使用pthread_detach的时候，希望调用Thread的detach方法。这样Thread的线程标志不会混乱。调用pthread_detach后，如果不调用pthread_exit方法，会导致线程结构有个8K的内存没有释放掉。默认情况下是没有detach的，此时，如果要释放线程的内存，需要在线程执行完成之后，不管是否调用了pthread_exit方法，都调用pthread_join方法阻塞销毁线程占用的那个8K内存。这也是我为何要将Thread封装起来的原因之一。我们有时候不想detach一个线程，这时候，我们就需要用join来释放，重复调用的话，会导致出现 fatal signal 6 的情况。 

备注2：
关于NDK 常见的出错信息意义：
fatal signal 4： 常见情况是方法没有返回值，比如一个返回int的方法，到最后没有return ret。
fatal signal 6：常见情况是pthread 线程类阻塞了，比如重复调用join方法，或者一个线程detach之后，然后又调用join就会出现这种情况
fatal signal 11：空指针出错。在多线程环境下，由于对象在另外一个线程释放调用，而该线程并没有停止，仍然在运行阶段，此时调用到该被释放的对象时，就会出现fatal signal 11 的错误。

其他的出错信息一般比较少见，至少本人接触到的NDK代码，还没遇到过其他出错信息。

好了，我们这里封装完了基础公共类之后，就可以愉快地编写C/C++代码了。
完整代码请参考本人的播放器项目：[CainPlayer](https://github.com/CainKernel/CainPlayer)

