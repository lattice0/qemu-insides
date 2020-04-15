Qemu main loop - part 2
================================================================================

The C main loop
--------------------------------------------------------------------------------
We're still at [util/main-loop.c#os_host_main_loop_wait](https://github.com/qemu/qemu/blob/stable-4.1/util/main-loop.c#L224)
So, with the timeout calculated from the step before, let's dive into this function:

```C
static int os_host_main_loop_wait(int64_t timeout)
{
    GMainContext *context = g_main_context_default();
    int ret;

    g_main_context_acquire(context);

    glib_pollfds_fill(&timeout);

    qemu_mutex_unlock_iothread();
    replay_mutex_unlock();

    ret = qemu_poll_ns((GPollFD *)gpollfds->data, gpollfds->len, timeout);

    replay_mutex_lock();
    qemu_mutex_lock_iothread();

    glib_pollfds_poll();

    g_main_context_release(context);

    return ret;
}
```

Now it's time to explain what a polling system is, and how Qemu implements it.

How would you write a program that listens for a socket connection? The first thing that might come to your mind is writing a while loop that checks indefintely if a socket file descriptor has data to be read. You might realize after a while, that this will make your program run all the time. You might then try to implement a delay function: you wait 50ms after each check, so you'll end doing only 20 checks per second, instead of millions. The problem with this approach is that 50ms is a lot of time, and so your progam will not be as responsive as you want, and also data might accumulate on the socket.

Operating systems prove you with a poll api. It's basically the same idea as before: you wait some milliseconds in every loop iteration, but with the advantage that your waiting function will return instantly if there's data in the socket. With this, your program will not do many checks per second, but you'll also be able to respond to them instantly!

Both macOS and Linux implement the POSIX api, so their polling functions are the same. For Windows it's a bit different, so that's why, instead of using the POSIX polling API, it uses `gpoll` from [Glib](https://developer.gnome.org/glib/), which handles the polling for each system. The function `qemu_poll_ns` actually calls `gpoll`, unless the macro `CONFIG_PPOLL` is defined. In this case, it uses `ppoll`, which we'll not cover.

As you can see, Qemu's main function is just a loop that polls file descriptors. That is, it runs a while loop that is able to respond instantly to events delivered through file descriptors. The function `qemu_poll_ns` returns instantly when there's something to poll, and `glib_pollfds_poll` does the actual polling. 

`glib_pollfds_poll` is defined at [util/main-loop.c#L212](https://github.com/qemu/qemu/blob/stable-4.1/util/main-loop.c#L212):

```C
static void glib_pollfds_poll(void)
{
    GMainContext *context = g_main_context_default();
    GPollFD *pfds = &g_array_index(gpollfds, GPollFD, glib_pollfds_idx);

    if (g_main_context_check(context, max_priority, pfds, glib_n_poll_fds)) {
        g_main_context_dispatch(context);
    }
}
```
