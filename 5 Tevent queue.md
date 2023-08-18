## 5 Tevent queue

调度程序及其处理程序可能无法在所有传入事件到达时尽快处理这些事件。处理这种情况的一种方法是通过在事件生成器和调度器之间的事件流中引入事件队列来缓冲接收到的事件。事件到达时会添加到队列中，调度程序会尽快将它们从队列的开头弹出。在 tevent 库中，它是类似的，但队列不是为任何事件自动设置的。必须故意创建队列，并且必须明确指出应遵循 FIFO 队列顺序的事件。在顺序处理对于成功完成任务至关重要的情况下，例如对于即将从缓冲区写入套接字的大量数据，创建这样的队列至关重要。tevent 库有自己的队列结构，在初始化并启动一次后即可使用。

### 5.1 创建队列

第一步也是最重要的一步是创建 tevent 队列（由结构 tevent_queue 表示），然后该队列将处于运行模式。

```C
struct tevent_queue* tevent_queue_create (TALLOC_CTX *mem_ctx, const char *name)
```

当程序从该函数返回时，分配的内存、设置析构函数和标记的队列都已完成，并且结构已准备好用条目填充。正在运行中停止和启动队列。如果您需要停止队列处理其条目，然后再次打开它，有几个函数可以实现这一目的：
- bool tevent_queue_stop()
- bool tevent_queue_start()

这些函数实际上只提供了一个变量的简单设置，该变量表示队列已停止/启动。返回值表示结果。

### 5.2 将请求添加到队列

事实上，Tevent 提供了 3 种将请求插入队列的可能方法。它们之间没有太大的区别，但在某些情况下，其中一种可能比另一种更适合和更受欢迎。

```C
bool tevent_queue_add(struct tevent_queue *queue,
                      struct tevent_context *ev,
                      struct tevent_req *req,
                      tevent_queue_trigger_fn_t trigger,
                      void *private_data)
```

这个调用是三个调用中最简单的一个。它只提供将请求添加到队列的操作是否成功的布尔验证。不可能从队列中额外删除项目，即只能解除分配整个 tevent 请求，这将导致触发析构函数处理，并将请求从队列中删除。

#### 5.2.1 扩展选项

以下两个函数都有一个共同的特性 —— 它们返回表示队列中项目的 tevent 队列条目结构。除了使用结构的指针来解除分配（这也会导致将其从队列中删除）之外，对该结构没有进一步的可能处理。不同之处在于，使用以下函数可以从队列中删除 tevent 请求，而无需释放它。前面的函数只能将 tevent 请求从内存中释放，从而在逻辑上也会将其从队列中删除。在 tevent 库的这一阶段，没有通过 API 对该结构进行其他利用。在使用 tevent 进行开发时，可以更容易地进行调试，这可以被认为是此返回指针的优点。

```C
struct tevent_queue_entry *tevent_queue_add_entry(struct tevent_queue *queue,
                                                  struct tevent_context *ev,
                                                  struct tevent_req *req,
                                                  tevent_queue_trigger_fn_t trigger,
                                                  void *private_data)
```

允许向队列中优化添加条目的功能是，首先检查没有项目的空队列。如果发现队列为空，则将省略向队列中插入条目的请求，并直接触发该请求。

```C
struct tevent_queue_entry *tevent_queue_add_optimize_empty(struct tevent_queue *queue,
                                                            struct tevent_context *ev,
                                                            struct tevent_req *req,
                                                            tevent_queue_trigger_fn_t trigger,
                                                            void *private_data)
```

当调用任何用于将项插入队列的函数时，可以省略第四个参数（触发器），而不是函数传递 NULL 指针。这种用法设置所谓的阻塞条目。这些条目，因为它们没有任何要激活的触发操作，所以只停留在它们的位置，直到它们被标记为由另一个函数完成。它们的目的是阻止队列中的其他项目被触发。

### 5.3 tevent 队列示例

```C
#include <stdio.h>
#include <unistd.h>
#include <tevent.h>
struct foo_state {
    int local_var;
    int x;
};
struct juststruct {
    TALLOC_CTX * ctx;
    struct tevent_context *ev;
    int y;
};
int created = 0;
static void timer_handler(struct tevent_context *ev, struct tevent_timer *te,
                           struct timeval current_time, void *private_data)
{
    // time event which after all sets request as done. Following item from
    // the queue  may be invoked.
    struct tevent_req *req = private_data;
    struct foo_state *stateX = tevent_req_data(req, struct foo_state);
    // processing some stuff
    printf("time_handler\n");
    tevent_req_done(req);
    talloc_free(req);
    printf("Request #%d set as done.\n", stateX->x);
}
static void trigger(struct tevent_req *req, void *private_data)
{
    struct juststruct *priv = tevent_req_callback_data (req, struct juststruct);
    struct foo_state *in = tevent_req_data(req, struct foo_state);
    struct timeval schedule;
    struct tevent_timer *tim;
    schedule = tevent_timeval_current_ofs(1, 0);
    printf("Processing request #%d\n", in->x);
    if (in->x % 3 == 0) {   // just example; third request does not contain
                            // any further operation and will be finished right
                            // away.
        tim = NULL;
    } else {
        tim = tevent_add_timer(priv->ev, req, schedule, timer_handler, req);
    }
    if (tim == NULL) {
            tevent_req_done(req);
            talloc_free(req);
            printf("Request #%d set as done.\n", in->x);
    }
}
struct tevent_req *foo_send(TALLOC_CTX *mem_ctx, struct tevent_context *ev,
                            const char *name, int num)
{
    struct tevent_req *req;
    struct foo_state *state;
    struct foo_state *in;
    struct tevent_timer *tim;
    printf("foo_send\n");
    req = tevent_req_create(mem_ctx, &state, struct foo_state);
    if (req == NULL) { // check for appropriate allocation
        tevent_req_error(req, 1);
        return NULL;
    }
    // exemplary filling of variables
    state->local_var = 1;
    state->x = num;
    return req;
}
static void foo_done(struct tevent_req *req) {
    enum tevent_req_state state;
    uint64_t err;
    if (tevent_req_is_error(req, &state, &err)) {
        printf("ERROR WAS SET %d\n", state);
        return;
    } else {
        // processing some stuff
        printf("Callback is done...\n");
    }
}
int main (int argc, char **argv)
{
    TALLOC_CTX *mem_ctx;
    struct tevent_req* req[6];
    struct tevent_req* tmp;
    struct tevent_context *ev;
    struct tevent_queue *fronta = NULL;
    struct juststruct *data;
    int ret;
    int i = 0;
    const char * const names[] = {
        "first", "second", "third", "fourth", "fifth"
    };
    printf("INIT\n");
    mem_ctx = talloc_new(NULL); //parent
    talloc_parent(mem_ctx);
    ev = tevent_context_init(mem_ctx);
    if (ev == NULL) {
        fprintf(stderr, "MEMORY ERROR\n");
        return EXIT_FAILURE;
    }
    // setting up queue
    fronta = tevent_queue_create(mem_ctx, "test_queue");
    tevent_queue_stop(fronta);
    tevent_queue_start(fronta);
    if (tevent_queue_running(fronta)) {
        printf ("Queue is runnning (length: %d)\n", tevent_queue_length(fronta));
    } else {
        printf ("Queue is not runnning\n");
    }
    data = talloc(ev, struct juststruct);
    data->ctx = mem_ctx;
    data->ev = ev;
    // create 4 requests
    for (i = 1; i < 5; i++) {
        req[i] = foo_send(mem_ctx, ev, names[i], i);
        tmp = req[i];
        if (req[i] == NULL) {
            fprintf(stderr, "Request error! %d \n", ret);
            break;
        }
        tevent_req_set_callback(req[i], foo_done, data);
        created++;
    }
    // add item to a queue
    tevent_queue_add(fronta, ev, req[1], trigger, data);
    tevent_queue_add(fronta, ev, req[2], trigger, data);
    tevent_queue_add(fronta, ev, req[3], trigger, data);
    tevent_queue_add(fronta, ev, req[4], trigger, data);
    printf("Queue length: %d\n", tevent_queue_length(fronta));
    while(tevent_queue_length(fronta) > 0) {
        tevent_loop_once(ev);
        printf("Queue: %d items left\n", tevent_queue_length(fronta));
    }
    talloc_free(mem_ctx);
    printf("FINISH\n");
    return EXIT_SUCCESS;
}
```