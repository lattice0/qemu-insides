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

Unfortunately it's too big to post here. 

...

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

Qemu runs all of this in a loop forever. So now we should check the [Main Loop]() section.

Let's now dive into `qemu_init`.

...

Let's take a look at `machine_run_board_init`:

```C
void machine_run_board_init(MachineState *machine)
{
    MachineClass *machine_class = MACHINE_GET_CLASS(machine);

    if (machine->ram_memdev_id) {
        Object *o;
        o = object_resolve_path_type(machine->ram_memdev_id,
                                     TYPE_MEMORY_BACKEND, NULL);
        machine->ram = machine_consume_memdev(machine, MEMORY_BACKEND(o));
    }

    if (machine->numa_state) {
        numa_complete_configuration(machine);
        if (machine->numa_state->num_nodes) {
            machine_numa_finish_cpu_init(machine);
        }
    }

    /* If the machine supports the valid_cpu_types check and the user
     * specified a CPU with -cpu check here that the user CPU is supported.
     */
    if (machine_class->valid_cpu_types && machine->cpu_type) {
        ObjectClass *class = object_class_by_name(machine->cpu_type);
        int i;

        for (i = 0; machine_class->valid_cpu_types[i]; i++) {
            if (object_class_dynamic_cast(class,
                                          machine_class->valid_cpu_types[i])) {
                /* The user specificed CPU is in the valid field, we are
                 * good to go.
                 */
                break;
            }
        }

        if (!machine_class->valid_cpu_types[i]) {
            /* The user specified CPU is not valid */
            error_report("Invalid CPU type: %s", machine->cpu_type);
            error_printf("The valid types are: %s",
                         machine_class->valid_cpu_types[0]);
            for (i = 1; machine_class->valid_cpu_types[i]; i++) {
                error_printf(", %s", machine_class->valid_cpu_types[i]);
            }
            error_printf("\n");

            exit(1);
        }
    }

    machine_class->init(machine);
}
```

See the `machine_class->init(machine);`?  It comes from `MachineClass *machine_class = MACHINE_GET_CLASS(machine);`. 
It gets the class for the current machine architecture you're in. For now, let's suppose it's `x86`. Then, this 'class' is actually
the one from x86.c, which is defined here with

```C
static const TypeInfo x86_machine_info = {
    .name = TYPE_X86_MACHINE,
    .parent = TYPE_MACHINE,
    .abstract = true,
    .instance_size = sizeof(X86MachineState),
    .instance_init = x86_machine_initfn,
    .class_size = sizeof(X86MachineClass),
    .class_init = x86_machine_class_init,
    .interfaces = (InterfaceInfo[]) {
         { TYPE_NMI },
         { }
    },
};
```

Function `x86_machine_class_init` is the one that inits this class
