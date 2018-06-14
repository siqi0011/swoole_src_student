# Swoole 源码学习记录

  工作五年多了前三年一直在做windows下面的c/c++，后面两年断断续续用php做一些web后端程序，后来因为工作要写一个硬件的socket服务器，本来打算用c/c++写的，一位学长建议我用swoole来搭建socket服务

器，swoole稳定高效的性能使我的开发变得简单、高效。

  因此，我希望更加深入的了解swoole的核心，学习他的架构设计拉。

  因此，我将不定期更新这一系列，将我在阅读、理解swoole源码过程中的心得体会记录下来，

也希望我的记录能帮助那些同样希望理解swoole源码的人。

# Swoole Memory  之Lock学习

php代码

  互斥锁

```
$lock = new swoole_lock(SWOOLE_MUTEX);
echo "[Master]create lock\n";
$lock->lock();
if (pcntl_fork() > 0)
{
    sleep(1);
    $lock->unlock();
} 
else
{
    echo "[Child] Wait Lock\n";
    $lock->lock();
    echo "[Child] Get Lock\n";
    $lock->unlock();
    exit("[Child] exit\n");
}
echo "[Master]release lock\n";
unset($lock);
sleep(1);
echo "[Master]exit\n";
```



swoole_lock.c 

swoole_lock是在扩展中注册好的类，用来给php调用，当php调用$lock = new swoole_lock(SWOOLE_MUTEX);

首先malloc一块内存当作swoole_lock对象的载体，然后从这个类的函数表中找到__construct函数调用拉，

然后判断switch type 等于SW_MUTEX---》

调用swMutex_create, 创建一个swLock 对象（结构体对象）拉---》

调用swoole_set_obj 把swLock锁对象的指针和 swoole_lock对象的指针做一个绑定拉。



当php调用对象的lock方法，流程转移到swoole_lock lock函数---->

调用swoole_get_obj 根据this指针从swoole_objects中取出对象绑定的swLock拉----》

然后调用swLock的lock函数---》

然后调用系统调用pthread_mutex_lock这样就完成了加锁拉。





总结：

​	学到了不少哈。

​	1） 很早就会c语言利用结构体+函数指针模拟的面向对象，不过这次看到实际应用拉。

​	2） 学习了怎么在php扩展中创建一个php类给php使用拉

​	3） 学习了enum的应用方法不错哈，swoole给每个枚举值定义的同时定义了一个宏哈,方便

​	4)   学会了pthread_mutex*系列系统调用的使用姿势， 首先init初始化互斥体，lock上锁，最后unlock解锁。。。

​	5）以上是我学习lock的流程，到这里拉，休息休息哈，希望大加多多指教。



```
static PHP_METHOD(swoole_lock, __construct)
{
    long type = SW_MUTEX;
    char *filelock;
    zend_size_t filelock_len = 0;
    int ret;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "|ls", &type, &filelock, &filelock_len) == FAILURE)
    {
        RETURN_FALSE;
    }

    swLock *lock = SwooleG.memory_pool->alloc(SwooleG.memory_pool, sizeof(swLock));
    if (lock == NULL)
    {
        zend_throw_exception(swoole_exception_class_entry_ptr, "global memory allocation failure.", SW_ERROR_MALLOC_FAIL TSRMLS_CC);
        RETURN_FALSE;
    }

    switch(type)
    {
#ifdef HAVE_RWLOCK
    case SW_RWLOCK:
        ret = swRWLock_create(lock, 1);
        break;
#endif
    case SW_FILELOCK:
        if (filelock_len <= 0)
        {
            zend_throw_exception(swoole_exception_class_entry_ptr, "filelock requires file name of the lock.", SW_ERROR_INVALID_PARAMS TSRMLS_CC);
            RETURN_FALSE;
        }
        int fd;
        if ((fd = open(filelock, O_RDWR | O_CREAT, 0666)) < 0)
        {
            zend_throw_exception_ex(swoole_exception_class_entry_ptr, errno TSRMLS_CC, "open file[%s] failed. Error: %s [%d]", filelock, strerror(errno), errno);
            RETURN_FALSE;
        }
        ret = swFileLock_create(lock, fd);
        break;
    case SW_SEM:
        ret = swSem_create(lock, IPC_PRIVATE);
        break;
#ifdef HAVE_SPINLOCK
    case SW_SPINLOCK:
        ret = swSpinLock_create(lock, 1);
        break;
#endif
    case SW_MUTEX:
    default:
        ret = swMutex_create(lock, 1);
        break;
    }
    if (ret < 0)
    {
        zend_throw_exception(swoole_exception_class_entry_ptr, "failed to create lock.", errno TSRMLS_CC);
        RETURN_FALSE;
    }
    swoole_set_object(getThis(), lock);
    RETURN_TRUE;
}
```



## 锁初始化函数

总共有六种哈 swRWLock_create， swMutex_create等每个类型的锁对应不同的

锁初始化函数拉，在成员函数__construct中调用拉。



```
int swMutex_create(swLock *lock, int use_in_process)
{
    int ret;
    bzero(lock, sizeof(swLock));
    lock->type = SW_MUTEX;
    pthread_mutexattr_init(&lock->object.mutex.attr);
    if (use_in_process == 1)
    {
        pthread_mutexattr_setpshared(&lock->object.mutex.attr, PTHREAD_PROCESS_SHARED);
    }
    if ((ret = pthread_mutex_init(&lock->object.mutex._lock, &lock->object.mutex.attr)) < 0)
    {
        return SW_ERR;
    }
    lock->lock = swMutex_lock;
    lock->unlock = swMutex_unlock;
    lock->trylock = swMutex_trylock;
    lock->free = swMutex_free;
    return SW_OK;
}

```



##  SW_LOCKS 锁的种类，swoole给每个枚举值还同时定义了一个对应名称的宏哈，方便了不少。



​	

```
enum SW_LOCKS
{
    SW_RWLOCK = 1,
#define SW_RWLOCK SW_RWLOCK
    SW_FILELOCK = 2,
#define SW_FILELOCK SW_FILELOCK
    SW_MUTEX = 3,
#define SW_MUTEX SW_MUTEX
    SW_SEM = 4,
#define SW_SEM SW_SEM
    SW_SPINLOCK = 5,
#define SW_SPINLOCK SW_SPINLOCK
    SW_ATOMLOCK = 6,
#define SW_ATOMLOCK SW_ATOMLOCK
};
```

​	

##    swLock 是一个用c语言结构体+函数指针实现的面向对象，

## 这里的lock unlock函数还使用了接口的设计哈，不管读写锁，

## 互斥锁都有这些共同的方法拉，优雅的设计哈。

```
typedef struct _swLock
{
	int type;
    union
    {
        swMutex mutex;
#ifdef HAVE_RWLOCK
        swRWLock rwlock;
#endif
#ifdef HAVE_SPINLOCK
        swSpinLock spinlock;
#endif
        swFileLock filelock;
        swSem sem;
        swAtomicLock atomlock;
    } object;

    int (*lock_rd)(struct _swLock *);
    int (*lock)(struct _swLock *);
    int (*unlock)(struct _swLock *);
    int (*trylock_rd)(struct _swLock *);
    int (*trylock)(struct _swLock *);
    int (*free)(struct _swLock *);
} swLock;
```



## mutex上锁，解锁函数

```
static PHP_METHOD(swoole_lock, lock)
{
    swLock *lock = swoole_get_object(getThis());
    SW_LOCK_CHECK_RETURN(lock->lock(lock));
}

static PHP_METHOD(swoole_lock, unlock)
{
    swLock *lock = swoole_get_object(getThis());
    SW_LOCK_CHECK_RETURN(lock->unlock(lock));
}
```



```
static int swMutex_lock(swLock *lock)
{
    return pthread_mutex_lock(&lock->object.mutex._lock);
}

static int swMutex_unlock(swLock *lock)
{
    return pthread_mutex_unlock(&lock->object.mutex._lock);
}
```



