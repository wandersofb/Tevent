## 3 Accessing data

### 3.1 使用 tevent 访问数据

tevent request（通常）与用于存储异步计算所需数据的结构一起创建。对于这些私有数据，tevent 库使用 void（泛型）指针，因此任何数据类型都可以非常简单地指向。然而，这种态度需要事先对将要处理的数据类型有明确和有保证的了解。私有数据可以有两种类型：与请求本身连接，或者作为单独的参数提供给回调。有必要区分这些类型，因为每种类型的数据访问方法略有不同。如何访问作为参数直接提供给回调的数据有两种可能性。区别在于返回的指针。在一种情况下，它是函数参数中指定的数据类型，在另一种情况中，返回 void*。

```C
void tevent_req_callback_data (struct tevent_req *req, #type)
void tevent_req_callback_data_void (struct tevent_req *req)
```

要获得严格绑定到请求的数据，此函数是唯一的直接过程。

```C
void *tevent_req_data (struct tevent_req *req, #type)
```

tevent 请求中的私有数据和作为参数移交的数据之间不同的两个调用的示例。

```C
#include <stdio.h>
#include <unistd.h>
#include <tevent.h>
struct foo_state {
    int x;
};
struct testA {
    int y;
};
static void foo_done(struct tevent_req *req) {
    // a->x contains 10 since it came from foo_send
    struct foo_state *a = tevent_req_data(req, struct foo_state);
    // b->y contains 9 since it came from run
    struct testA *b = tevent_req_callback_data(req, struct testA);
    // c->y contains 9 since it came from run we just used a different way
    // of getting it.
    struct testA *c = (struct testA *)tevent_req_callback_data_void(req);
    printf("a->x: %d\n", a->x);
    printf("b->y: %d\n", b->y);
    printf("c->y: %d\n", c->y);
}
struct tevent_req * foo_send(TALLOC_CTX *mem_ctx, struct tevent_context *event_ctx) {
    printf("_send\n");
    struct tevent_req *req;
    struct foo_state *state;
    req = tevent_req_create(event_ctx, &state, struct foo_state);
    state->x = 10;
    return req;
}
static void run(struct tevent_context *ev, struct tevent_timer *te,
                struct timeval current_time, void *private_data) {
    struct tevent_req *req;
    struct testA *tmp = talloc(ev, struct testA);
    // Note that we did not use the private data passed in
    tmp->y = 9;
    req = foo_send(ev, ev);
    tevent_req_set_callback(req, foo_done, tmp);
    tevent_req_done(req);
}
int main (int argc, char **argv) {
    struct tevent_context *event_ctx;
    struct testA *data;
    TALLOC_CTX *mem_ctx;
    struct tevent_timer *time_event;
    mem_ctx = talloc_new(NULL); //parent
    if (mem_ctx == NULL)
        return EXIT_FAILURE;
    event_ctx = tevent_context_init(mem_ctx);
    if (event_ctx == NULL)
        return EXIT_FAILURE;
    data = talloc(mem_ctx, struct testA);
    data->y = 11;
    time_event = tevent_add_timer(event_ctx,
                                  mem_ctx,
                                  tevent_timeval_current(),
                                  run,
                                  data);
    if (time_event == NULL) {
        fprintf(stderr, " FAILED\n");
        return EXIT_FAILURE;
    }
    tevent_loop_once(event_ctx);
    talloc_free(mem_ctx);
    printf("Quit\n");
    return EXIT_SUCCESS;
}
```

Output of this example is:

```C
a->x: 10
b->y: 9
c->y: 9
```