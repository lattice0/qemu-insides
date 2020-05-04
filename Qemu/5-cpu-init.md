CPU/Machine init - part 5
================================================================================

Something
--------------------------------------------------------------------------------


Q35 CPU
--------------------------------------------------------------------------------
On [hw/i386/pc_q35.c](https://github.com/qemu/qemu/blob/17e1e49814096a3daaa8e5a73acd56a0f30bdc18/hw/i386/pc_q35.c#L115), you´ find the `pc_q35_init` function, which initializes a LOT of devices for us:

```C
static void pc_q35_init(MachineState *machine)
{
    PCMachineState *pcms = PC_MACHINE(machine);
    PCMachineClass *pcmc = PC_MACHINE_GET_CLASS(pcms);
    X86MachineState *x86ms = X86_MACHINE(machine);
    Q35PCIHost *q35_host;
    PCIHostState *phb;
    PCIBus *host_bus;
    PCIDevice *lpc;
    DeviceState *lpc_dev;
    BusState *idebus[MAX_SATA_PORTS];
    ISADevice *rtc_state;
    MemoryRegion *system_io = get_system_io();
    MemoryRegion *pci_memory;
    MemoryRegion *rom_memory;
    MemoryRegion *ram_memory;
    GSIState *gsi_state;
    ISABus *isa_bus;
    int i;
    ICH9LPCState *ich9_lpc;
    PCIDevice *ahci;
    ram_addr_t lowmem;
    DriveInfo *hd[MAX_SATA_PORTS];
    MachineClass *mc = MACHINE_GET_CLASS(machine);

    /* Check whether RAM fits below 4G (leaving 1/2 GByte for IO memory
     * and 256 Mbytes for PCI Express Enhanced Configuration Access Mapping
     * also known as MMCFG).
     * If it doesn't, we need to split it in chunks below and above 4G.
     * In any case, try to make sure that guest addresses aligned at
     * 1G boundaries get mapped to host addresses aligned at 1G boundaries.
     */
    if (machine->ram_size >= 0xb0000000) {
        lowmem = 0x80000000;
    } else {
        lowmem = 0xb0000000;
    }

    /* Handle the machine opt max-ram-below-4g.  It is basically doing
     * min(qemu limit, user limit).
     */
    if (!x86ms->max_ram_below_4g) {
        x86ms->max_ram_below_4g = 4 * GiB;
    }
    if (lowmem > x86ms->max_ram_below_4g) {
        lowmem = x86ms->max_ram_below_4g;
        if (machine->ram_size - lowmem > lowmem &&
            lowmem & (1 * GiB - 1)) {
            warn_report("There is possibly poor performance as the ram size "
                        " (0x%" PRIx64 ") is more then twice the size of"
                        " max-ram-below-4g (%"PRIu64") and"
                        " max-ram-below-4g is not a multiple of 1G.",
                        (uint64_t)machine->ram_size, x86ms->max_ram_below_4g);
        }
    }

    if (machine->ram_size >= lowmem) {
        x86ms->above_4g_mem_size = machine->ram_size - lowmem;
        x86ms->below_4g_mem_size = lowmem;
    } else {
        x86ms->above_4g_mem_size = 0;
        x86ms->below_4g_mem_size = machine->ram_size;
    }

    if (xen_enabled()) {
        xen_hvm_init(pcms, &ram_memory);
    }

    x86_cpus_init(x86ms, pcmc->default_cpu_version);

    kvmclock_create();

    /* pci enabled */
    if (pcmc->pci_enabled) {
        pci_memory = g_new(MemoryRegion, 1);
        memory_region_init(pci_memory, NULL, "pci", UINT64_MAX);
        rom_memory = pci_memory;
    } else {
        pci_memory = NULL;
        rom_memory = get_system_memory();
    }

    pc_guest_info_init(pcms);

    if (pcmc->smbios_defaults) {
        /* These values are guest ABI, do not change */
        smbios_set_defaults("QEMU", "Standard PC (Q35 + ICH9, 2009)",
                            mc->name, pcmc->smbios_legacy_mode,
                            pcmc->smbios_uuid_encoded,
                            SMBIOS_ENTRY_POINT_21);
    }

    /* allocate ram and load rom/bios */
    if (!xen_enabled()) {
        pc_memory_init(pcms, get_system_memory(),
                       rom_memory, &ram_memory);
    }

    /* create pci host bus */
    q35_host = Q35_HOST_DEVICE(qdev_create(NULL, TYPE_Q35_HOST_DEVICE));

    object_property_add_child(qdev_get_machine(), "q35", OBJECT(q35_host), NULL);
    object_property_set_link(OBJECT(q35_host), OBJECT(ram_memory),
                             MCH_HOST_PROP_RAM_MEM, NULL);
    object_property_set_link(OBJECT(q35_host), OBJECT(pci_memory),
                             MCH_HOST_PROP_PCI_MEM, NULL);
    object_property_set_link(OBJECT(q35_host), OBJECT(get_system_memory()),
                             MCH_HOST_PROP_SYSTEM_MEM, NULL);
    object_property_set_link(OBJECT(q35_host), OBJECT(system_io),
                             MCH_HOST_PROP_IO_MEM, NULL);
    object_property_set_int(OBJECT(q35_host), x86ms->below_4g_mem_size,
                            PCI_HOST_BELOW_4G_MEM_SIZE, NULL);
    object_property_set_int(OBJECT(q35_host), x86ms->above_4g_mem_size,
                            PCI_HOST_ABOVE_4G_MEM_SIZE, NULL);
    /* pci */
    qdev_init_nofail(DEVICE(q35_host));
    phb = PCI_HOST_BRIDGE(q35_host);
    host_bus = phb->bus;
    /* create ISA bus */
    lpc = pci_create_simple_multifunction(host_bus, PCI_DEVFN(ICH9_LPC_DEV,
                                          ICH9_LPC_FUNC), true,
                                          TYPE_ICH9_LPC_DEVICE);

    object_property_add_link(OBJECT(machine), PC_MACHINE_ACPI_DEVICE_PROP,
                             TYPE_HOTPLUG_HANDLER,
                             (Object **)&pcms->acpi_dev,
                             object_property_allow_set_link,
                             OBJ_PROP_LINK_STRONG, &error_abort);
    object_property_set_link(OBJECT(machine), OBJECT(lpc),
                             PC_MACHINE_ACPI_DEVICE_PROP, &error_abort);

    /* irq lines */
    gsi_state = pc_gsi_create(&x86ms->gsi, pcmc->pci_enabled);

    ich9_lpc = ICH9_LPC_DEVICE(lpc);
    lpc_dev = DEVICE(lpc);
    for (i = 0; i < GSI_NUM_PINS; i++) {
        qdev_connect_gpio_out_named(lpc_dev, ICH9_GPIO_GSI, i, x86ms->gsi[i]);
    }
    pci_bus_irqs(host_bus, ich9_lpc_set_irq, ich9_lpc_map_irq, ich9_lpc,
                 ICH9_LPC_NB_PIRQS);
    pci_bus_set_route_irq_fn(host_bus, ich9_route_intx_pin_to_irq);
    isa_bus = ich9_lpc->isa_bus;

    pc_i8259_create(isa_bus, gsi_state->i8259_irq);

    if (pcmc->pci_enabled) {
        ioapic_init_gsi(gsi_state, "q35");
    }

    if (tcg_enabled()) {
        x86_register_ferr_irq(x86ms->gsi[13]);
    }

    assert(pcms->vmport != ON_OFF_AUTO__MAX);
    if (pcms->vmport == ON_OFF_AUTO_AUTO) {
        pcms->vmport = xen_enabled() ? ON_OFF_AUTO_OFF : ON_OFF_AUTO_ON;
    }

    /* init basic PC hardware */
    pc_basic_device_init(isa_bus, x86ms->gsi, &rtc_state, !mc->no_floppy,
                         (pcms->vmport != ON_OFF_AUTO_ON), pcms->pit_enabled,
                         0xff0104);

    /* connect pm stuff to lpc */
    ich9_lpc_pm_init(lpc, x86_machine_is_smm_enabled(x86ms));

    if (pcms->sata_enabled) {
        /* ahci and SATA device, for q35 1 ahci controller is built-in */
        ahci = pci_create_simple_multifunction(host_bus,
                                               PCI_DEVFN(ICH9_SATA1_DEV,
                                                         ICH9_SATA1_FUNC),
                                               true, "ich9-ahci");
        idebus[0] = qdev_get_child_bus(&ahci->qdev, "ide.0");
        idebus[1] = qdev_get_child_bus(&ahci->qdev, "ide.1");
        g_assert(MAX_SATA_PORTS == ahci_get_num_ports(ahci));
        ide_drive_get(hd, ahci_get_num_ports(ahci));
        ahci_ide_create_devs(ahci, hd);
    } else {
        idebus[0] = idebus[1] = NULL;
    }

    if (machine_usb(machine)) {
        /* Should we create 6 UHCI according to ich9 spec? */
        ehci_create_ich9_with_companions(host_bus, 0x1d);
    }

    if (pcms->smbus_enabled) {
        /* TODO: Populate SPD eeprom data.  */
        pcms->smbus = ich9_smb_init(host_bus,
                                    PCI_DEVFN(ICH9_SMB_DEV, ICH9_SMB_FUNC),
                                    0xb100);
        smbus_eeprom_init(pcms->smbus, 8, NULL, 0);
    }

    pc_cmos_init(pcms, idebus[0], idebus[1], rtc_state);

    /* the rest devices to which pci devfn is automatically assigned */
    pc_vga_init(isa_bus, host_bus);
    pc_nic_init(pcmc, isa_bus, host_bus);

    if (machine->nvdimms_state->is_enabled) {
        nvdimm_init_acpi_state(machine->nvdimms_state, system_io,
                               x86ms->fw_cfg, OBJECT(pcms));
    }
}
```
This function is very important, as it initializes a machine for the x86 architecture. We're gonna understand its pieces one by one. 

Let's start with `PCMachineState` and `PCMachineClass`. We already seen those before: `PCMachineClass` is a subclass of `MachineClass`, so it extends its functions. `PCMachineState` is the state of this class, as we already seen before.

On [include/hw/i386/pc.h#L29](https://github.com/qemu/qemu/blob/stable-4.1/include/hw/i386/pc.h#L29), we can see the declaration of `PCMachineState`:

```C
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

`MachineState` is its parent. I'll give a brief description on some important structures that we'll later review in detail:


`HotplugHandler *acpi_dev`: link to ACPI PM device that performs ACPI hotplug handling. If we look at the [include/hw/hotplug.h](https://github.com/qemu/qemu/blob/stable-4.1/include/hw/hotplug.h#L26) file for this, we find its struct and class definitions, with methods of type `hotplug_fn` which handle hotplugging of devices. 
But what's ACPI? 

"[ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface), or Advanced Configuration and Power Interface (ACPI), provides an open standard that operating systems can use to discover and configure computer hardware components, to perform power management by (for example) putting unused components to sleep, and to perform status monitoring."

"Internally, ACPI advertises the available components and their functions to the operating system kernel using instruction lists ("methods") provided through the system firmware (Unified Extensible Firmware Interface (UEFI) or BIOS), which the kernel parses. ACPI then executes the desired operations written in ACPI Machine Language (such as the initialization of hardware components) using an embedded minimal virtual machine."

Turns out most of the information about your computer will be gathered on boot time through ACPI, but it looks like it also supports hot plugging of PCI devices, which can also provide ACPI information. I couldn't find much about it online so I'm not really sure.

`ISADevice* rtc`: I think it stands for the old [ISA BUS](https://en.wikipedia.org/wiki/Industry_Standard_Architecture).

`PCIBus *bus`: of course, PCI bus, which is important and we'll probably handle later. 

`FWCfgState *fw_cfg`: looks like something related to NVRAM (Non-Volatile Random Access Memory), where `fw_cfg` stands for "firmware configuration". 

`qemu_irq *gsi`: just a pointer to IRQState: `typedef struct IRQState *qemu_irq`, which holds an `qemu_irq_handler`, which is the function pointer `typedef void (*qemu_irq_handler)(void *opaque, int n, int level)`. Interrupts are central in understanding CPUs. I'm not sure right now how this handler works but we'll probably understand it in the future.

`PFlashCFI01 *flash[2]`: on its .h file it says `NOR flash devices`.

`uint64_t max_ram_below_4g`: this variable is explained in the `pc_q35_init` function that this article intends to analyze. We're gonna see it later but this is a variable setted by the option `max-ram-below-4g` passed to qemu on startup.

`bool smbus_enabled`: "The System Management Bus (abbreviated to SMBus or SMB) is a single-ended simple two-wire bus for the purpose of lightweight communication. Most commonly it is found in computer motherboards for communication with the power source for ON/OFF instructions."

`ram_addr_t below_4g_mem_size, above_4g_mem_size`: just `typedef uint64_t ram_addr_t` in case `XEN_BACKEND` is configured or `typedef uintptr_t ram_addr_t` otherwise. The variables 
`below_4g_mem_size` and `above_4g_mem_size` are explained also in the `pc_q35_init` but they split the RAM in 2 parts.


`uint64_t numa_nodes`: TBD

`uint64_t *node_mem`: TBD

`AddressSpace *ioapic_as`: Address space used by IOAPIC device. What's IOAPIC? "The Intel I/O Advanced Programmable Interrupt Controller is used to distribute external interrupts in a more advanced manner than that of the standard 8259 PIC. With the I/O APIC, interrupts can be distributed to physical or logical (clusters of) processors and can be prioritized. Each I/O APIC typically handles 24 external interrupts."

The class `PCMachineClass` almost doesn't have custom data structures, only `int`s and `bool`s.

