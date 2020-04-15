Qemu main function - part 1
================================================================================

The C main function
--------------------------------------------------------------------------------

So, [here it is](https://github.com/qemu/qemu/blob/17e1e49814096a3daaa8e5a73acd56a0f30bdc18/softmmu/main.c), the function that bootstraps every C program:

```C
#include "qemu/osdep.h"
#include "qemu-common.h"
#include "sysemu/sysemu.h"

#ifdef CONFIG_SDL
#if defined(__APPLE__) || defined(main)
#include <SDL.h>
int main(int argc, char **argv)
{
    return qemu_main(argc, argv, NULL);
}
#undef main
#define main qemu_main
#endif
#endif /* CONFIG_SDL */

#ifdef CONFIG_COCOA
#undef main
#define main qemu_main
#endif /* CONFIG_COCOA */

int main(int argc, char **argv, char **envp)
{
    qemu_init(argc, argv, envp);
    qemu_main_loop();
    qemu_cleanup();

    return 0;
}
```
Let's dive into `qemu_init`.

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
