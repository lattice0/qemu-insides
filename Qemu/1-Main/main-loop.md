Qemu main loop - part 1
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
