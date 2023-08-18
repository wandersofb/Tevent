## 6 Tevent with threads

为了将 tevent 与线程一起使用，您必须首先了解如何在线程程序中使用 talloc 库。有关使用 talloc 的更多信息，请访问 talloc 网站，那里有教程和文档。
如果 tevent 下文结构是从 NULL、线程安全的 talloc 上下文中进行 talloc 的，那么在线程程序中使用它是安全的。在进行任何 talloc 调用之前，必须从初始程序线程调用函数 talloc_disable_null_tracking()，以确保 talloc 是线程安全的。
每个线程都必须创建自己的 tevent 上下文结构，如下 tevent_context_init(NULL) 所示，并且线程之间不能共享 talloc 内存上下文。
以这种方式使用 tevent 的独立线程可以通过将数据写入由另一个线程上的 tevent 上下文监视的文件描述符来进行通信。例如（简化后没有错误处理）：

```C
Main thread:
main()
{
        talloc_disable_null_tracking();
        struct tevent_context *master_ev = tevent_context_init(NULL);
        void *mem_ctx = talloc_new(master_ev);
        // Create file descriptor to monitor.
        int pipefds[2];
        pipe(pipefds);
        struct tevent_fd *fde = tevent_add_fd(master_ev,
                                mem_ctx,
                                pipefds[0], // read side of pipe
                                TEVENT_FD_READ,
                                pipe_read_handler, // callback function
                                private_data_pointer);
        // Create sub thread, pass pipefds[1] write side of pipe to it.
        // The above code not shown here..
        // Process events.
        tevent_loop_wait(master_ev);
        // Cleanup if loop exits.
        talloc_free(master_ev);
}
```

当子线程写入 pipefds[1] 时，函数 pipe_read_handler() 将在主线程中调用。

### 6.1 复杂的用途

在线程程序中使用事件库的一种流行方法是允许子线程从另一个线程的事件循环异步调度 tevent_immediate 函数调用。这可以由 tevent 的基本功能和隔离机制构建而成，但 tevent 也附带了一些实用程序功能，只要您了解将线程与 talloc 和 tevent 一起使用所带来的限制，这些功能就会使这变得更容易。

要允许 tevent 上下文从另一个线程接收异步 tevent_immediate 函数回调，请通过调用

```C
struct tevent_thread_proxy *tevent_thread_proxy_create(
                struct tevent_context *dest_ev_ctx);
```

此函数分配内部数据结构以允许异步回调作为结构体 tevent_context\*的 talloc 子级，并返回可传递给另一个线程的结构体teevent_thread_proxy\*。

当您接收完异步回调后，只需 talloc_free 结构体tevent_thread_proxy\*，或 talloc_free tevent_context\*，即可释放所使用的资源。

要在另一个线程的 tevent 循环上调度一个线程对异步 tevent_immediate 函数的调用，请使用

```C
void tevent_thread_proxy_schedule(struct tevent_thread_proxy *tp,
                                struct tevent_immediate **pp_im,
                                tevent_immediate_handler_t handler,
                                void **pp_private_data);
```

此函数会导致函数 handler() 作为 tevent_immediate 回调从创建结构 tevent_thread_proxy\* 的线程的事件循环中调用（因此拥有的结构 tevent_context\* 应该是长寿命的，而不是在被拆除的过程中）。

此处使用的结构体 tevent_thread_proxy 对象是目标线程的事件上下文的子对象。因此，必须使用外部同步机制来确保在 tevent_thread_proxy_schedule() 调用时目标对象仍在使用中。在下面的示例中，通信的请求/响应性质确保了这一点。

传递到此函数的结构体 tevent_immediate \*\*pp_im 应该是分配在此线程本地的 talloc 上下文上的结构体 tevent_immediate\*，并且将通过 talloc_move 重新生成，以归结构体 teevent_thread_proxy \*tp。所有在成功调度 tevent_immediate 调用时，\*pp_im将被设置为NULL。

handler() 将作为一个普通的 tevent_immediate 回调从创建结构 tevent_thread_proxy\* 的目标事件循环的结构 tevent_context\*调用。

从这些函数返回并不意味着处理程序已被调用，只是意味着它已被计划在目标事件循环中调用。

由于调用线程不等待回调被调度并在目标线程上运行，因此这是一个即发即弃调用。如果希望确认 handler() 被成功调用，则必须确保它以某种方式回复调用者。

由于此调用的异步性质，传递给目标线程的参数的性质有一些重构。如果不需要参数，只需将 NULL 作为 void \*\*pp_private_data的值传递即可。

如果希望在线程之间传递指向数据的指针，则它必须是指向 talloced 指针的指针，该指针不是 talloc 池的一部分，并且不能附加析构函数。指向的内存的所有权将从调用线程传递到 tevent 库，如果接收线程没有将其重新分配到自己的上下文，则在调用处理程序后将释放该内存。

成功后，\*pp_private 将为 NULL，表示 talloc 内存所有权已被移动。

在实践中，对于事件循环中线程之间的消息传递，这些限制并不是很繁重。

在不同线程上的 tevent 循环之间建立请求 - 应答对的最简单方法是使用应答 tevent_thread_proxy_schedule() 调用来回传递内存的参数块。

以下是一个示例（为了简单起见，没有进行错误检查）：

```C
------------------------------------------------
// Master thread.
main()
{
        // Make talloc thread-safe.
        talloc_disable_null_tracking();
        // Create the master event context.
        struct tevent_context *master_ev = tevent_context_init(NULL);
        // Create the master thread proxy to allow it to receive
        // async callbacks from other threads.
        struct tevent_thread_proxy *master_tp =
                        tevent_thread_proxy_create(master_ev);
        // Create sub-threads, passing master_tp in
        // some way to them.
        // This code not shown..
        // Process events.
        // Function master_callback() below
        // will be invoked on this thread on
        // master_ev event context.
        tevent_loop_wait(master_ev);
        // Cleanup if loop exits.
        talloc_free(master_ev);
}
// Data passed between threads.
struct reply_state {
        struct tevent_thread_proxy *reply_tp;
        pthread_t thread_id;
        bool *p_finished;
};
// Callback Called in child thread context.
static void thread_callback(struct tevent_context *ev,
                                struct tevent_immediate *im,
                                void *private_ptr)
{
        // Move the ownership of what private_ptr
        // points to from the tevent library back to this thread.
        struct reply_state *rsp =
                talloc_get_type_abort(private_ptr, struct reply_state);
        talloc_steal(ev, rsp);
         *rsp->p_finished = true;
        // im will be talloc_freed on return from this call.
        // but rsp will not.
}
// Callback Called in master thread context.
static void master_callback(struct tevent_context *ev,
                                struct tevent_immediate *im,
                                void *private_ptr)
{
        // Move the ownership of what private_ptr
        // points to from the tevent library to this thread.
        struct reply_state *rsp =
                talloc_get_type_abort(private_ptr, struct reply_state);
        talloc_steal(ev, rsp);
        printf("Callback from thread %s\n", thread_id_to_string(rsp->thread_id));
        /* Now reply to the thread ! */
        tevent_thread_proxy_schedule(rsp->reply_tp,
                                &im,
                                thread_callback,
                                &rsp);
        // Note - rsp and im are now NULL as the tevent library
        // owns the memory.
}
// Child thread.
static void *thread_fn(void *private_ptr)
{
        struct tevent_thread_proxy *master_tp =
                talloc_get_type_abort(private_ptr, struct tevent_thread_proxy);
        bool finished = false;
        int ret;
        // Create our own event context.
        struct tevent_context *ev = tevent_context_init(NULL);
        // Create the local thread proxy to allow us to receive
        // async callbacks from other threads.
        struct tevent_thread_proxy *local_tp =
                        tevent_thread_proxy_create(master_ev);
        // Setup the data to send.
        struct reply_state *rsp = talloc(ev, struct reply_state);
        rsp->reply_tp = local_tp;
        rsp->thread_id = pthread_self();
        rsp->p_finished = &finished;
        // Create the immediate event to use.
        struct tevent_immediate *im = tevent_create_immediate(ev);
        // Call the master thread.
        tevent_thread_proxy_schedule(master_tp,
                                &im,
                                master_callback,
                                &rsp);
        // Note - rsp and im are now NULL as the tevent library
        // owns the memory.
        // Wait for the reply.
        while (!finished) {
                tevent_loop_once(ev);
        }
        // Cleanup.
        talloc_free(ev);
        return NULL;
}
```

请注意，这不一定是一个主要的子线程通信。任何有权访问另一个调用 tevent_thread_proxy_create() 的线程的结构 tevent_thread_proxy\* 指针的线程都可以发送异步 tevent_immediat e请求。
但请记住，必须使用外部同步来确保在调用 tevent_thread_proxy_schedule() 时目标结构 tevent_tthread_proxy\* 对象存在，否则将导致不可修复的崩溃。