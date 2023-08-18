## 2 Tevent events

好的，在阅读了上一章之后，我们可以开始做一些有用的事情。因此，创建事件的方式对于所有类型都是相似的 —— 信号、文件描述符、时间或即时事件。一开始，最好了解一些在 tevent 库中设置的 typedef，它们为每个回调指定参数。这些回调是：

- tevent_timer_handler_t()
- tevent_immediate_handler_t()
- tevent_signal_handler_t()
- tevent_fd_handler_t()

根据它们的名称，很明显，为了创建时间事件的回调，将使用tevent_timer_handler_t。

如何引入注册事件和设置回调的最佳方法是示例，因此下面是描述所有类型事件的示例。

### 2.1 时间事件

此示例显示如何设置一个事件，该事件将以 2 秒为间隔重复一分钟（将触发 30 次）。超过此限制后，事件循环将结束，所有内存资源将被释放。这只是描述重复活动的示例，在 foo 函数中没有执行任何有用的操作

```c
#include <stdio.h>
#include <unistd.h>
#include <tevent.h>
#include <sys/time.h>
struct state {
     struct timeval endtime;
     int counter;
     TALLOC_CTX *ctx;
};
static void callback(struct tevent_context *ev, struct tevent_timer *tim,
                     struct timeval current_time, void *private_data)
{
    struct state *data = talloc_get_type_abort(private_data, struct state);
    struct tevent_timer *time_event;
    struct timeval schedule;
    printf("Data value: %d\n", data->counter);
    data->counter += 1; // increase counter
    // if time has not reached its limit, set another event
    if (tevent_timeval_compare(&current_time, &(data->endtime)) < 0) {
        // do something
        // set repeat with delay 2 seconds
        schedule = tevent_timeval_current_ofs(2, 0);
        time_event = tevent_add_timer(ev, data->ctx, schedule, callback, data);
        if (time_event == NULL) { // error ...
            fprintf(stderr, "MEMORY PROBLEM\n");
            return;
        }
    } else {
        // time limit exceeded
    }
}
int main(void)  {
    struct tevent_context *event_ctx;
    TALLOC_CTX *mem_ctx;
    struct tevent_timer *time_event;
    struct timeval schedule;
    mem_ctx = talloc_new(NULL); // parent
    event_ctx = tevent_context_init(mem_ctx);
    struct state *data = talloc(mem_ctx, struct state);
    schedule = tevent_timeval_current_ofs(2, 0); // +2 second time value
    data->endtime = tevent_timeval_add(&schedule, 60, 0); // one minute time limit
    data->ctx = mem_ctx;
    data->counter = 0;
    // add time event
    time_event = tevent_add_timer(event_ctx, mem_ctx, schedule, callback, data);
    if (time_event == NULL) {
        fprintf(stderr, "FAILED\n");
        return EXIT_FAILURE;
    }
    tevent_loop_wait(event_ctx);
    talloc_free(mem_ctx);
    return EXIT_SUCCESS;
}
```

变量计数器仅用于对触发的函数数量进行计数。这里列出了 tevent 为随时间工作提供的所有可用功能的列表及其描述。没有必要对这些函数进行更详细的查看，因为它们的目的和用法非常简单明了。

### 2.2 即时事件

正如其名称所示，这些事件被激活并立即执行。这意味着这类事件的优先级高于其他事件（信号事件除外）。因此，如果注册了大量事件，然后启动 tevent 循环，那么所有即时事件都将在其他事件之前触发。除了其他即时事件（和信号事件），因为它们也按计划顺序进行处理。信号具有最高优先级，因此它们被优先处理。因此，“立即”一词可能与字典中“没有延迟的事情”的定义不完全一致，而是在所有先前的立即事件之后“尽快”。

对于创建即时事件，有一个小的不同之处，那就是创建此类事件分两步完成。一个表示创建（内存分配），第二个表示在某个 tevent 上下文中注册为事件。

```c
struct tevent_immediate *run(TALLOC_CTX* mem_ctx,
                             struct tevent_context event_ctx,
                             void * data)
{
    struct tevent_immediate *im;
    im = tevent_create_immediate(mem_ctx);
    if (im == NULL) {
        return NULL;
    }
    tevent_schedule_immediate(im, event_ctx, foo, data);
    return im;
}
```

可以编译和运行的示例，表示立即事件的创建。

```c
#include <stdio.h>
#include <unistd.h>
#include <tevent.h>
struct info_struct {
    int counter;
};
static void foo(struct tevent_context *ev, struct tevent_immediate *im,
                void *private_data)
{
    struct info_struct *data = talloc_get_type_abort(private_data, struct info_struct);
    printf("Data value: %d\n", data->counter);
}
int main (void) {
    struct tevent_context *event_ctx;
    TALLOC_CTX *mem_ctx;
    struct tevent_immediate *im;
    printf("INIT\n");
    mem_ctx = talloc_new(NULL);
    event_ctx = tevent_context_init(mem_ctx);
    struct info_struct *data = talloc(mem_ctx, struct info_struct);
    // setting up private data
    data->counter = 1;
    // first immediate event
    im = tevent_create_immediate(mem_ctx);
    if (im == NULL) {
        fprintf(stderr, "FAILED\n");
        return EXIT_FAILURE;
    }
    tevent_schedule_immediate(im, event_ctx, foo, data);
    tevent_loop_wait(event_ctx);
    talloc_free(mem_ctx);
    return 0;
}
```

### 2.3 信号事件

这是标准 C 库函数 signal() 或 sigaction() 的替代方法。区别这些处理信号的方法的主要区别在于，它们为运行程序的不同时间间隔设置了处理程序。

虽然处理信号的标准 C 库方法在大多数情况下提供了足够的工具，但它们不足以处理 tevent 循环中的信号。可能需要在 tevent 循环中不间断地完成某些 tevent 请求。如果在 tevent 循环进行的时候向程序发送了信号，那么标准信号处理程序将不会在同一位置向应用程序返回处理，并且将永远退出 tevent 循环。在这种情况下，tevent 信号处理程序提供了处理这些信号的可能性，方法是将它们与应用程序的其他部分屏蔽开来，而不退出循环，这样其他事件仍然可以处理。

Tevent 还提供了一个控制函数，使我们能够验证是否可以通过 Tevent 处理信号，该函数在 Tevent 库中定义，并返回一个布尔值，显示验证结果。

```c
bool tevent_signal_support (struct tevent_context *ev)
```

检查信号支持是不必要的，但如果不能保证，这是一个很好的简单控制，可以防止程序出现意外行为或故障。当然，这样的测试不必每次都要创建信号处理程序，而只需在程序的初始化过程中的一开始就运行。之后，只需适应出现的每一种情况。

```c
#include <stdio.h>
#include <tevent.h>
#include <signal.h>
static void handler(struct tevent_context *ev,
                    struct tevent_signal *se,
                    int signum,
                    int count,
                    void *siginfo,
                    void *private_data)
{
    // Do something usefull
    printf("handling signal...\n");
    exit(EXIT_SUCCESS);
}
int main (void)
{
    struct tevent_context *event_ctx;
    TALLOC_CTX *mem_ctx;
    struct tevent_signal *sig;
    mem_ctx = talloc_new(NULL); //parent
    if (mem_ctx == NULL) {
        fprintf(stderr, "FAILED\n");
        return EXIT_FAILURE;
    }
    event_ctx = tevent_context_init(mem_ctx);
    if (event_ctx == NULL) {
        fprintf(stderr, "FAILED\n");
        return EXIT_FAILURE;
    }
    if (tevent_signal_support(event_ctx)) {
        // create signal event
        sig = tevent_add_signal(event_ctx, mem_ctx, SIGINT, 0, handler, NULL);
        if (sig == NULL) {
            fprintf(stderr, "FAILED\n");
            return EXIT_FAILURE;
        }
        tevent_loop_wait(event_ctx);
    }
    talloc_free(mem_ctx);
    return EXIT_SUCCESS;
}
```

### 2.4 文件描述符事件

对文件描述符上的事件的支持主要用于套接字通信，但它也可以完美地与标准流（stdin、stdout、stderr）配合使用。通过与文件描述符异步工作，可以在处理 I/O 操作中进行切换。这种能力可能随着 I/O 操作次数的增加而增加，并且这种重叠导致吞吐量的提高。

tevent API 中还包含了其他几个与处理文件描述符相关的函数（tevent 中定义的函数太多，因此本文仅对其中一些函数进行了全面描述。其余函数的声明可以在库的网站上轻松找到，也可以直接从源代码中找到）：
- tevent_fd_set_close_fn（）- 可以在结构 tevent-fd 被释放的时刻。
- tevent_fd_set_auto_close（）- 调用此函数可以简化文件描述符的维护，因为它指示 tevent 在即将释放 tevent-fd 结构时关闭相应的文件描述符。
- tevent_fd_get_flags（）- 返回在与该 tevent-fd 结构连接的文件描述符上设置的标志。
- tevent_fd_set_flags（）- 在事件的文件描述符上设置指定的标志。

```c
static void close_fd(struct tevent_context *ev, struct tevent_fd *fd_event,
                     int fd, void *private_data)
{
    // processing when fd_event is freed
}
struct static void handler(struct tevent_context *ev,
                           struct tevent_fd *fde,
                           uint16_t flags,
                           void *private_data)
{
    // handling event; reading from a file descriptor
    tevent_fd_set_close_fn (fd_event, close_fd);
}
int run(TALLOC_CTX *mem_ctx, struct tevent_context *event_ctx,
        int fd, uint16_t flags, char *buffer)
{
    struct tevent_fd* fd_event = NULL;
    if (flags & TEVENT_FD_READ) {
        fd_event = tevent_add_fd(event_ctx,
                                 mem_ctx,
                                 fd,
                                 flags,
                                 handler,
                                 buffer);
    }
    if (fd_event == NULL) {
        // error handling
    }
    return tevent_loop_once();
}
```