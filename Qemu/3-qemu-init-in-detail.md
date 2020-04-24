Qemu init process in detail
================================================================================

Init process for some 
--------------------------------------------------------------------------------

We're still inside `int main(int argc, char **argv, char **envp)`. We're gonna analyze the CPU init for now.

QEMU has two main machines: Q35 and i440fx. On the main function, there's the line [vl.c/#3857](https://github.com/qemu/qemu/blob/stable-4.1/vl.c#L3857) where it selects the main machine:
```C
machine_class = select_machine();
```

so lets see, in the same file [vl.c/#2568](https://github.com/qemu/qemu/blob/stable-4.1/vl.c#L2568), what it does:

```C
static MachineClass *select_machine(void)
{
    GSList *machines = object_class_get_list(TYPE_MACHINE, false);
    MachineClass *machine_class = find_default_machine(machines);
    const char *optarg;
    QemuOpts *opts;
    Location loc;

    loc_push_none(&loc);

    opts = qemu_get_machine_opts();
    qemu_opts_loc_restore(opts);

    optarg = qemu_opt_get(opts, "type");
    if (optarg) {
        machine_class = machine_parse(optarg, machines);
    }

    if (!machine_class) {
        error_report("No machine specified, and there is no default");
        error_printf("Use -machine help to list supported machines\n");
        exit(1);
    }

    loc_pop(&loc);
    g_slist_free(machines);
    return machine_class;
}
```

It reads the option "type" from the command "machine". If no machine is specified, it uses the default one from `find_default_machine(machines)`, where `machines` is a `glist` (an array implementation from Glib).

Then the instantiation occurs here:

```C
current_machine = MACHINE(object_new(object_class_get_name(
                          OBJECT_CLASS(machine_class))));
```

Let's expand those macros:

```C
OBJECT_CHECK(MachineState, (object_new(object_class_get_name( ((ObjectClass *)(machine_class))))), TYPE_MACHINE)
```

The `machine_class` object is converted to its `ObjectClass` (the class that all classes derive from). Then `object_class_get_name` accesses its type:

```C
const char *object_class_get_name(ObjectClass *klass)
{
    return klass->type->name;
}
```
Turns out that every `ObjectClass` has a Type (`typedef struct TypeImpl *Type`), where `TypeImpl` is:

```C
struct TypeImpl
{
    const char *name;
    //...
```

So, given the name of the class, now `Object *object_new(const char *typename);` creates an object from it, and the `MACHINE` macro casts it to the `MachineState` type by doing `OBJECT_CHECK`:

```C
#define OBJECT_CHECK(type,obj,name) ((type *)object_dynamic_cast_assert(OBJECT(obj), (name), __FILE__, __LINE__, __func__))
```

After all this code, remember that we're still in the same function (`main`). Ignoring most of the code for now, we reach [vl.c#L4348](https://github.com/qemu/qemu/blob/stable-4.1/vl.c#L4348) which is simply the line 

```C
machine_run_board_init(current_machine);
```

Finally, our `MachineState* current_machine` is going to be used to initiate the MachineClass:

```C
void machine_run_board_init(MachineState *machine)
{
    MachineClass *machine_class = MACHINE_GET_CLASS(machine);

    numa_complete_configuration(machine);
    if (nb_numa_nodes) {
        machine_numa_finish_cpu_init(machine);
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

So now we have a `MachineState` in current_machine. It's a very important struct. Let's see what it does:

```C
struct MachineState {
    /*< private >*/
    Object parent_obj;
    Notifier sysbus_notifier;

    /*< public >*/

    char *accel;
    bool kernel_irqchip_allowed;
    bool kernel_irqchip_required;
    bool kernel_irqchip_split;
    int kvm_shadow_mem;
    char *dtb;
    char *dumpdtb;
    int phandle_start;
    char *dt_compatible;
    bool dump_guest_core;
    bool mem_merge;
    bool usb;
    bool usb_disabled;
    bool igd_gfx_passthru;
    char *firmware;
    bool iommu;
    bool suppress_vmdesc;
    bool enforce_config_section;
    bool enable_graphics;
    char *memory_encryption;
    DeviceMemoryState *device_memory;

    ram_addr_t ram_size;
    ram_addr_t maxram_size;
    uint64_t   ram_slots;
    const char *boot_order;
    char *kernel_filename;
    char *kernel_cmdline;
    char *initrd_filename;
    const char *cpu_type;
    AccelState *accelerator;
    CPUArchIdList *possible_cpus;
    CpuTopology smp;
    struct NVDIMMState *nvdimms_state;
};
```

It certainly has some important parts of our CPU as you can see by the names. For example, look at `accel`. It is setted in the qemu init function we reviwed before () rigth here: [vl.c#L3544](https://github.com/qemu/qemu/blob/stable-4.1/vl.c#L3544). The options are `kvm` (fast, kernel based virtualization) or `tcg` (slow, userspace CPU emulation).

We can assume for now that the other properties are setted in the init process or later, depending on configs passed and possibly other things (you can actually find most of them being setted in [vl.c/#main](https://github.com/qemu/qemu/blob/stable-4.1/vl.c#L2857)). The names are kinda self explanatory: most of them are `bool` so it indicates if the feature is active or not. Let's take a look on the less common ones.

`CPUArchIdList` is a list of `CPUArchId` objects, take a look at their explanations from [include/hw/boards.h#L81](https://github.com/qemu/qemu/blob/stable-4.1/include/hw/boards.h#L81):

```C
/**
 * CPUArchIdList:
 * @len - number of @CPUArchId items in @cpus array
 * @cpus - array of present or possible CPUs for current machine configuration
 */
typedef struct {
    int len;
    CPUArchId cpus[0];
} CPUArchIdList;

/**
 * CPUArchId:
 * @arch_id - architecture-dependent CPU ID of present or possible CPU
 * @cpu - pointer to corresponding CPU object if it's present on NULL otherwise
 * @type - QOM class name of possible @cpu object
 * @props - CPU object properties, initialized by board
 * #vcpus_count - number of threads provided by @cpu object
 */
typedef struct {
    uint64_t arch_id;
    int64_t vcpus_count;
    CpuInstanceProperties props;
    Object *cpu;
    const char *type;
} CPUArchId;
```

This list `CPUArchIdList *possible_cpus;` can be initialized by many portions of the code. Just as an example, [hw/core/machine.c#L1098](https://github.com/qemu/qemu/blob/stable-4.1/hw/core/machine.c#L1098), which is inside our already knwon `machine_run_board_init()` function, does this initialization in [hw/core/machine.c#L1042](https://github.com/qemu/qemu/blob/stable-4.1/hw/core/machine.c#L1042):

```C
static void machine_numa_finish_cpu_init(MachineState *machine)
{
    int i;
    bool default_mapping;
    GString *s = g_string_new(NULL);
    MachineClass *mc = MACHINE_GET_CLASS(machine);
    const CPUArchIdList *possible_cpus = mc->possible_cpu_arch_ids(machine);
```

where `possible_cpu_arch_ids` is defined, for x86 architecture, on [hw/i386/pc.c#L2944](https://github.com/qemu/qemu/blob/stable-4.1/hw/i386/pc.c#L2944):

```C
mc->possible_cpu_arch_ids = pc_possible_cpu_arch_ids;
```

where `pc_possible_cpu_arch_ids` is on [hw/i386/pc.c#L2861](https://github.com/qemu/qemu/blob/stable-4.1/hw/i386/pc.c#L2861):

```C
static const CPUArchIdList *pc_possible_cpu_arch_ids(MachineState *ms)
{
    PCMachineState *pcms = PC_MACHINE(ms);
    int i;
    unsigned int max_cpus = ms->smp.max_cpus;

    if (ms->possible_cpus) {
        /*
         * make sure that max_cpus hasn't changed since the first use, i.e.
         * -smp hasn't been parsed after it
        */
        assert(ms->possible_cpus->len == max_cpus);
        return ms->possible_cpus;
    }

    ms->possible_cpus = g_malloc0(sizeof(CPUArchIdList) +
                                  sizeof(CPUArchId) * max_cpus);
    ms->possible_cpus->len = max_cpus;
    for (i = 0; i < ms->possible_cpus->len; i++) {
        X86CPUTopoInfo topo;

        ms->possible_cpus->cpus[i].type = ms->cpu_type;
        ms->possible_cpus->cpus[i].vcpus_count = 1;
        ms->possible_cpus->cpus[i].arch_id = x86_cpu_apic_id_from_index(pcms, i);
        x86_topo_ids_from_apicid(ms->possible_cpus->cpus[i].arch_id,
                                 pcms->smp_dies, ms->smp.cores,
                                 ms->smp.threads, &topo);
        ms->possible_cpus->cpus[i].props.has_socket_id = true;
        ms->possible_cpus->cpus[i].props.socket_id = topo.pkg_id;
        if (pcms->smp_dies > 1) {
            ms->possible_cpus->cpus[i].props.has_die_id = true;
            ms->possible_cpus->cpus[i].props.die_id = topo.die_id;
        }
        ms->possible_cpus->cpus[i].props.has_core_id = true;
        ms->possible_cpus->cpus[i].props.core_id = topo.core_id;
        ms->possible_cpus->cpus[i].props.has_thread_id = true;
        ms->possible_cpus->cpus[i].props.thread_id = topo.smt_id;
    }
    return ms->possible_cpus;
}
```
You can see that this function is dependent of architecture. For the x86 architecture it sets these values.

A line worth noticing is `PCMachineState *pcms = PC_MACHINE(ms);`. Let's have a look on that structure on [include/hw/i386/pc.h#L29](https://github.com/qemu/qemu/blob/stable-4.1/include/hw/i386/pc.h#L29):

```C
/**
 * PCMachineState:
 * @acpi_dev: link to ACPI PM device that performs ACPI hotplug handling
 * @boot_cpus: number of present VCPUs
 * @smp_dies: number of dies per one package
 */
struct PCMachineState {
    /*< private >*/
    MachineState parent_obj;

    /* <public> */

    /* State for other subsystems/APIs: */
    Notifier machine_done;

    /* Pointers to devices and objects: */
    HotplugHandler *acpi_dev;
    ISADevice *rtc;
    PCIBus *bus;
    FWCfgState *fw_cfg;
    qemu_irq *gsi;
    PFlashCFI01 *flash[2];

    /* Configuration options: */
    uint64_t max_ram_below_4g;
    OnOffAuto vmport;
    OnOffAuto smm;

    bool acpi_build_enabled;
    bool smbus_enabled;
    bool sata_enabled;
    bool pit_enabled;

    /* RAM information (sizes, addresses, configuration): */
    ram_addr_t below_4g_mem_size, above_4g_mem_size;

    /* CPU and apic information: */
    bool apic_xrupt_override;
    unsigned apic_id_limit;
    uint16_t boot_cpus;
    unsigned smp_dies;

    /* NUMA information: */
    uint64_t numa_nodes;
    uint64_t *node_mem;

    /* Address space used by IOAPIC device. All IOAPIC interrupts
     * will be translated to MSI messages in the address space. */
    AddressSpace *ioapic_as;
};
```

It looks like there is a subclass for `Machine` called `PCMachine`. Indeed we find it at [include/hw/i386/pc.h#L99](https://github.com/qemu/qemu/blob/stable-4.1/include/hw/i386/pc.h#L99):

```C
 	typedef struct PCMachineClass {
    /*< private >*/
    MachineClass parent_class;

    /*< public >*/

    /* Device configuration: */
    bool pci_enabled;
    bool kvmclock_enabled;
    const char *default_nic_model;

    /* Compat options: */

    /* Default CPU model version.  See x86_cpu_set_default_version(). */
    int default_cpu_version;

    /* ACPI compat: */
    bool has_acpi_build;
    bool rsdp_in_ram;
    int legacy_acpi_table_size;
    unsigned acpi_data_size;

    /* SMBIOS compat: */
    bool smbios_defaults;
    bool smbios_legacy_mode;
    bool smbios_uuid_encoded;

    /* RAM / address space compat: */
    bool gigabyte_align;
    bool has_reserved_memory;
    bool enforce_aligned_dimm;
    bool broken_reserved_end;

    /* TSC rate migration: */
    bool save_tsc_khz;
    /* generate legacy CPU hotplug AML */
    bool legacy_cpu_hotplug;

    /* use DMA capable linuxboot option rom */
    bool linuxboot_dma_enabled;

    /* use PVH to load kernels that support this feature */
    bool pvh_enabled;

    /* Enables contiguous-apic-ID mode */
    bool compat_apic_id_mode;
} PCMachineClass;
```

So just as `MachineClass` has its `MachineState`, the subclass `PCMachineClass` has its state `PCMachineState`. You can see this pattern throughout the entire Qemu source code: a class called `SomethingClass`, and its state called `SomethingState`.

It makes sense to make a subclass for `MachineClass` because this code is located at `hw/i386`. So we can theorize that each CPU has its own subclass with its own features defined.

