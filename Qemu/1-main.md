Qemu main function - part 1
================================================================================

The C main function
--------------------------------------------------------------------------------

So, at [vl.c](https://github.com/qemu/qemu/blob/stable-4.1/vl.c#L2857), we find the function that bootstraps every C program:

```C
int main(int argc, char **argv, char **envp)
{
    //...
```

Unfortunately it's too big to post here, we're going to analyze it in parts. The bigger part of the main function is to parse and store the command line options, as this snippet shows:

```C
    case QEMU_OPTION_enable_kvm:
        olist = qemu_find_opts("machine");
        qemu_opts_parse_noisily(olist, "accel=kvm", false);
        break;
    case QEMU_OPTION_M:
    case QEMU_OPTION_machine:
        olist = qemu_find_opts("machine");
        opts = qemu_opts_parse_noisily(olist, optarg, true);
        if (!opts) {
            exit(1);
        }
        break;
        case QEMU_OPTION_no_kvm:
        olist = qemu_find_opts("machine");
        qemu_opts_parse_noisily(olist, "accel=tcg", false);
        break;
```
After that, some init functions for other things are called, we're gonna deal with them later.


Let's dive into the more important part, which is the main loop `main_loop()` found in this [line](https://github.com/qemu/qemu/blob/stable-4.1/vl.c#L4473), which is defined in the same file and only does this:

```C
static void main_loop(void)
{
#ifdef CONFIG_PROFILER
    int64_t ti;
#endif
    while (!main_loop_should_exit()) {
#ifdef CONFIG_PROFILER
        ti = profile_getclock();
#endif
        main_loop_wait(false);
#ifdef CONFIG_PROFILER
        dev_time += profile_getclock() - ti;
#endif
    }
}
```

As you can see, it checks all the time if it should exist. Otherwise, the loop continues. `main_loop_should_exit()` is just a function that calls a bunch of other functions that return true if qemu should shutdown, pause, reset, etc. The important part is in the `main_loop_wait` function, which is defined in [util/main-loop.c](https://github.com/qemu/qemu/blob/stable-4.1/util/main-loop.c#L488):

```C
void main_loop_wait(int nonblocking)
{
    MainLoopPoll mlpoll = {
        .state = MAIN_LOOP_POLL_FILL,
        .timeout = UINT32_MAX,
        .pollfds = gpollfds,
    };
    int ret;
    int64_t timeout_ns;

    if (nonblocking) {
        mlpoll.timeout = 0;
    }

    /* poll any events */
    g_array_set_size(gpollfds, 0); /* reset for new iteration */
    /* XXX: separate device handlers from system ones */
    notifier_list_notify(&main_loop_poll_notifiers, &mlpoll);

    if (mlpoll.timeout == UINT32_MAX) {
        timeout_ns = -1;
    } else {
        timeout_ns = (uint64_t)mlpoll.timeout * (int64_t)(SCALE_MS);
    }

    timeout_ns = qemu_soonest_timeout(timeout_ns,
                                      timerlistgroup_deadline_ns(
                                          &main_loop_tlg));

    ret = os_host_main_loop_wait(timeout_ns);
    mlpoll.state = ret < 0 ? MAIN_LOOP_POLL_ERR : MAIN_LOOP_POLL_OK;
    notifier_list_notify(&main_loop_poll_notifiers, &mlpoll);

    /* CPU thread can infinitely wait for event after
       missing the warp */
    qemu_start_warp_timer();
    qemu_clock_run_all_timers();
}
```

Qemu runs all of this in a loop forever. Now that you have an overview of the qemu event loop, we can start explaining the qemu init process in details.