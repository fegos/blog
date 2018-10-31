---
title: Nodejs之EventLoop
date: 2018-05-15
tags: nodejs
author: swx
---

### 初识EventLoop
EventLoop是一个程序结构，用于等待、发送消息和事件。

事件循环的职责，就是不断得等待事件的发生，然后将这个事件的所有处理器，以它们订阅这个事件的时间顺序，依次执行。当这个事件的所有处理器都被执行完毕之后，事件循环就会开始继续等待下一个事件的触发，不断往复。

![image](https://note.youdao.com/yws/api/personal/file/WEB36bfba2beaca697929ad575a619f5f88?method=download&shareKey=75c7e9f235e0adaf39668432e519b866)

### 从WebServer理解EventLoop
Node JS构成的web服务器和传统的多线程服务器不一样，Node JS只有一个进程处理请求和响应。

Node JS处理请求如下：
1. client发送request到server
2. server将requst放入名为EventQueue的队列
3. EventLoop从EventQueue中取出request
4. 如果request含有IO任务或者其他耗时，则从内部线程池中获取线程去处理该请求，一切处理完成后，将response发送回EventLoop
5. 如果request中不含有耗时任务，则EventLoop会直接处理，将response发送给client。


![image](https://note.youdao.com/yws/api/personal/file/WEBfd1cd7fffbabbcf4ed4a394c7cfa5b29?method=download&shareKey=aed35c468e885549ac91f2f0a0123c1d)

```
public class EventLoop {
    while(true){
        if(Event Queue receives a JavaScript Function Call){
            ClientRequest request = EventQueue.getClientRequest();
            If(request requires BlokingIO or takes more computation time)
                Assign request to Thread T1
            Else
                Process and Prepare response
        }
    }
} 
```

#### 优点
+ 对于多并发无压力
+ 不会随着请求的增多而迅速增大开销
+ 使用的线程比较少，所以占用的资源也比较少

### EventLoop vs Worker Pool
EventLoop 执行non-blocking的异步任务，一般为JavaScript的callback。

Worker Pool在libuv中实现，Worker Pool中执行“expensive”任务，包含操作系统没提供异步处理的任务，比如I/O，数据库访问等，还有部分比较消耗CPU的任务。

+ I/O集中的任务
    + DNS : dns.lookup(), dns.lookupService().
    + FileSystem
+ CPU集中的任务
    + Zlib
    + Crypto


### EventLoop各个阶段分析
当Node启动的时候会初始化Event Loop，在初始化Event Loop的时候会执行一段传入进来的脚本，然后开始事件循环。

事件循环的大体顺序如下：

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

#### 理解要点：

1. event loop 会分阶段执行，每个阶段都会有各自的FIFO队列
2. 到达每个阶段后会一直执行，直到当前阶段队列为空，或者执行的callback达到了最大限度
3. timers阶段涉及到的有setTimeout和setInterval
4. I/O callback阶段执行除了close、timer、setImmediate相关回调的所有回调
5. poll会获取新的I/O事件并执行，当队列为空的时候，会检测timer队列，如果timer对了不为空，并且已经到执行时间，则会执行timer。node有可能阻塞在这个阶段。
6. check阶段执行setImmediate的回调
7. close阶段执行所有的关闭回调，如socket的关闭
8. process.nextTick比较特殊，在每个阶段的末尾执行。
9. 官方建议使用setImediate代替process.nextTick。

#### timer阶段

timer会指定一个阈值，当达到这个阈值的时候，就会去执行对应的callback，然而实际执行的过程中，timer什么时候执行是由poll阶段控制的。

```
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

这里EventLoop进入poll阶段的时候，由于队列为空（I/O需要95ms，还没进行完），所以会阻塞在这里，等待95ms的timer，在90ms的时候，I/O执行完了，将callback加入了poll队列里面，开始执行I/O的callback，这个callback耗时10ms，执行完成后，会去检测timer，发现这时候timer已经可以执行了，这时候开始执行timer，所以真正执行timer是在105ms的时候，而不是100ms。

#### I/O阶段
执行类似于TCP error的操作，比如TCP的连接接收到`ECONNREFUSED`，就会将该callback放入这个队列去执行。

#### poll阶段
执行到期的timer，然后处理poll queue中的event。

+ 进入poll阶段的时候，如果没有timer
    + poll队列不为空，则执行poll队列中的callback
    + poll队列为空
        + 如果存在setImmediate()的回调，则会结束poll阶段，进入到check阶段
        + 如果不存在setImmediate()的回调，则会一直阻塞在这里，直到有callback加入队列

+ poll队列为空的时候，如果有timer
    + poll队列为空的时候，检测timer队列，如果有超时的，则会wrap back to timer阶段去执行

#### check阶段

当poll阶段处于空闲状态的时候，会立即执行setImmediate()的callback。执行完后，如果队列为空，会回到poll阶段。

#### close阶段
socket或者handle突然关闭(e.g. socket.destroy())，close event会在这个阶段去发送，否则会通过process.nextTick()发送close event。

#### setImmediate() vs setTimeout()
+ setImmediate 设计在poll阶段完成时执行，即check阶段
+ setTimeout 设计在poll阶段为空闲时，且设定时间到达后执行

二者的调用顺序取决于当前event loop的上下文，如果他们在异步I/O callback之外调用，其执行先后顺序是不确定的。如果在I/O callback内调用，则永远都是setImmediate()先执行。

不在I/O callback内执行

```
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});

```

```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

在I/O callback内调用

```
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

#### process.nextTick()
process.nextTick()会在每个阶段执行。

使用process有时候会带来便利：

```
let bar;

// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall(callback) { callback(); }

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {
  // since someAsyncApiCall has completed, bar hasn't been assigned any value
  console.log('bar', bar); // undefined
});

bar = 1;
```

```
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

创建服务器的时候，使用了process.nextTick();

```
const server = net.createServer(() => {}).listen(8080);
server.on('listening', () => {});
```
当端口传过去的时候，会立马绑定，这时候会发出listening事件，然而这时候.on('listening') callback还没有设置，这时候就需要在process.nextTick()中去发送listening事件。

### 从node源码分析
入口文件`src/node_main.cc`

```
int main(int argc, char *argv[]) {
#if defined(__linux__)
  char** envp = environ;
  while (*envp++ != nullptr) {}
  Elf_auxv_t* auxv = reinterpret_cast<Elf_auxv_t*>(envp);
  for (; auxv->a_type != AT_NULL; auxv++) {
    if (auxv->a_type == AT_SECURE) {
      node::linux_at_secure = auxv->a_un.a_val;
      break;
    }
  }
#endif
  // Disable stdio buffering, it interacts poorly with printf()
  // calls elsewhere in the program (e.g., any logging from V8.)
  setvbuf(stdout, nullptr, _IONBF, 0);
  setvbuf(stderr, nullptr, _IONBF, 0);
  return node::Start(argc, argv);
}
#endif
```
接收参数，对参数进行初步解析，传给node::Start函数。

`src/node.cc`
```
node::Start(int argc, char** argv) {
//....
  
    Init(&argc, const_cast<const char**>(argv), &exec_argc, &exec_argv);

    const int exit_code =
      Start(uv_default_loop(), argc, argv, exec_argc, exec_argv);
//...
return exit_code;
}
```
构造uv\_default\_loop()，也就是EventLoop。
`deps/uv/src/uv-common.c`
```

static uv_loop_t default_loop_struct;
static uv_loop_t* default_loop_ptr;

uv_loop_t* uv_default_loop(void) {
  if (default_loop_ptr != NULL)
    return default_loop_ptr;

  if (uv_loop_init(&default_loop_struct))
    return NULL;

  default_loop_ptr = &default_loop_struct;
  return default_loop_ptr;
}

```
`uv_loop_t`的结构：
`deps/uv/include/uv.h`
```
typedef struct uv_loop_s uv_loop_t;
struct uv_loop_s {
  /* User data - use this for whatever. */
  void* data;
  /* Loop reference counting. */
  unsigned int active_handles;
  void* handle_queue[2];
  void* active_reqs[2];
  /* Internal flag to signal loop stop. */
  unsigned int stop_flag;
  UV_LOOP_PRIVATE_FIELDS
};


```
`deps/uv/include/uv-unix.h`
```
#define UV_LOOP_PRIVATE_FIELDS                                                \
  unsigned long flags;                                                        \
  int backend_fd;                                                             \
  void* pending_queue[2];                                                     \
  void* watcher_queue[2];                                                     \
  uv__io_t** watchers;                                                        \
  unsigned int nwatchers;                                                     \
  unsigned int nfds;                                                          \
  void* wq[2];                                                                \
  uv_mutex_t wq_mutex;                                                        \
  uv_async_t wq_async;                                                        \
  uv_rwlock_t cloexec_lock;                                                   \
  uv_handle_t* closing_handles;                                               \
  void* process_handles[2];                                                   \
  void* prepare_handles[2];                                                   \
  void* check_handles[2];                                                     \
  void* idle_handles[2];                                                      \
  void* async_handles[2];                                                     \
  void (*async_unused)(void);  /* TODO(bnoordhuis) Remove in libuv v2. */     \
  uv__io_t async_io_watcher;                                                  \
  int async_wfd;                                                              \
  struct {                                                                    \
    void* min;                                                                \
    unsigned int nelts;                                                       \
  } timer_heap;                                                               \
  uint64_t timer_counter;                                                     \
  uint64_t time;                                                              \
  int signal_pipefd[2];                                                       \
  uv__io_t signal_io_watcher;                                                 \
  uv_signal_t child_watcher;                                                  \
  int emfile_fd;                                                              \
  UV_PLATFORM_LOOP_FIELDS                                                     \
```

```
node::Start(uv_loop_t* event_loop,
                 int argc, const char* const* argv,
                 int exec_argc, const char* const* exec_argv) {
    IsolateData isolate_data(
        isolate,
        event_loop,
        v8_platform.Platform(),
        allocator.zero_fill_field());
    if (track_heap_objects) {
      isolate->GetHeapProfiler()->StartTrackingHeapObjects(true);
    }
    exit_code = Start(isolate, &isolate_data, argc, argv, exec_argc, exec_argv);
}
```
IsolateData是V8中的概念，代表一个独立的虚拟机，对应一个或多个线程。但同一时刻 只能被一个线程进入。所有的 Isolate 彼此之间是完全隔离的, 它们不能够有任何共享的资源。如果不显示创建 Isolate, 会自动创建一个默认的 Isolate。

`src/node.cc`
```
inline int Start(Isolate* isolate, IsolateData* isolate_data,
                 int argc, const char* const* argv,
                 int exec_argc, const char* const* exec_argv) {
//...
    {
    SealHandleScope seal(isolate);
    bool more;
    PERFORMANCE_MARK(&env, LOOP_START);
    do {
      uv_run(env.event_loop(), UV_RUN_DEFAULT);

      v8_platform.DrainVMTasks(isolate);

      more = uv_loop_alive(env.event_loop());
      if (more)
        continue;

      EmitBeforeExit(&env);

      // Emit `beforeExit` if the loop became alive either after emitting
      // event, or after running some callbacks.
      more = uv_loop_alive(env.event_loop());
    } while (more == true);
    PERFORMANCE_MARK(&env, LOOP_EXIT);
  }
//...
}
```

`deps/uv/src/unix/core.c`
```
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

`deps/uv/src/unix/core.c`
```
static int uv__run_pending(uv_loop_t* loop) {
  QUEUE* q;
  QUEUE pq;
  uv__io_t* w;

  if (QUEUE_EMPTY(&loop->pending_queue))
    return 0;

  QUEUE_MOVE(&loop->pending_queue, &pq);

  while (!QUEUE_EMPTY(&pq)) {
    q = QUEUE_HEAD(&pq);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);
    w = QUEUE_DATA(q, uv__io_t, pending_queue);
    w->cb(loop, w, POLLOUT);
  }

  return 1;
}
```

`deps/uv/src/unix/linux-core.c`
```
void uv__io_poll(uv_loop_t* loop, int timeout) {
  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
   
    e.events = w->pevents;
    e.data = w->fd;

    if (w->events == 0)
      op = UV__EPOLL_CTL_ADD;
    else
      op = UV__EPOLL_CTL_MOD;

    /* XXX Future optimization: do EPOLL_CTL_MOD lazily if we stop watching
     * events, skip the syscall and squelch the events after epoll_wait().
     */
    if (uv__epoll_ctl(loop->backend_fd, op, w->fd, &e)) {
      if (errno != EEXIST)
        abort();

      assert(op == UV__EPOLL_CTL_ADD);

      /* We've reactivated a file descriptor that's been watched before. */
      if (uv__epoll_ctl(loop->backend_fd, UV__EPOLL_CTL_MOD, w->fd, &e))
        abort();
    }

    w->events = w->pevents;
  }
  
  
}
```

### 异同
表现形式一样

实现不一样

node中事件循环依靠libuv引擎，多个队列，没有微任务和宏任务的划分。
### 参考文献

https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick

https://yjhjstz.gitbooks.io/deep-into-node/chapter2/chapter2-0.html
