Qemu OOP in C - part 3
================================================================================

How Object Oriented Programming language concepts are implemented in C on Qemu project
--------------------------------------------------------------------------------

Most of the 'classes' (structs) used in Qemu are defined in [qemu/typedefs.h](). Basically they're typedef'ed to have the same name as the class name, probably just in case you need to rename it in the future:

```C
#ifndef QEMU_TYPEDEFS_H
#define QEMU_TYPEDEFS_H

/* A load of opaque types so that device init declarations don't have to
   pull in all the real definitions.  */

/* Please keep this list in case-insensitive alphabetical order */
typedef struct AdapterInfo AdapterInfo;
typedef struct AddressSpace AddressSpace;
typedef struct AioContext AioContext;
typedef struct AnnounceTimer AnnounceTimer;
typedef struct BdrvDirtyBitmap BdrvDirtyBitmap;
typedef struct BdrvDirtyBitmapIter BdrvDirtyBitmapIter;
typedef struct BlockBackend BlockBackend;
typedef struct BlockBackendRootState BlockBackendRootState;
typedef struct BlockDriverState BlockDriverState;
typedef struct BusClass BusClass;
typedef struct BusState BusState;
typedef struct Chardev Chardev;
typedef struct CompatProperty CompatProperty;
typedef struct CoMutex CoMutex;
typedef struct CPUAddressSpace CPUAddressSpace;
typedef struct CPUState CPUState;
typedef struct DeviceListener DeviceListener;
typedef struct DeviceState DeviceState;
typedef struct DirtyBitmapSnapshot DirtyBitmapSnapshot;
typedef struct DisplayChangeListener DisplayChangeListener;
typedef struct DriveInfo DriveInfo;
typedef struct Error Error;
typedef struct EventNotifier EventNotifier;
typedef struct FlatView FlatView;
typedef struct FWCfgEntry FWCfgEntry;
typedef struct FWCfgIoState FWCfgIoState;
typedef struct FWCfgMemState FWCfgMemState;
typedef struct FWCfgState FWCfgState;
typedef struct HVFX86EmulatorState HVFX86EmulatorState;
typedef struct I2CBus I2CBus;
typedef struct I2SCodec I2SCodec;
typedef struct IOMMUMemoryRegion IOMMUMemoryRegion;
typedef struct ISABus ISABus;
typedef struct ISADevice ISADevice;
typedef struct IsaDma IsaDma;
typedef struct MACAddr MACAddr;
typedef struct MachineClass MachineClass;
typedef struct MachineState MachineState;
typedef struct MemoryListener MemoryListener;
typedef struct MemoryMappingList MemoryMappingList;
typedef struct MemoryRegion MemoryRegion;
typedef struct MemoryRegionCache MemoryRegionCache;
typedef struct MemoryRegionSection MemoryRegionSection;
typedef struct MigrationIncomingState MigrationIncomingState;
typedef struct MigrationState MigrationState;
typedef struct Monitor Monitor;
typedef struct MonitorDef MonitorDef;
typedef struct MSIMessage MSIMessage;
typedef struct NetClientState NetClientState;
typedef struct NetFilterState NetFilterState;
typedef struct NICInfo NICInfo;
typedef struct NodeInfo NodeInfo;
typedef struct NumaNodeMem NumaNodeMem;
typedef struct ObjectClass ObjectClass;
typedef struct PCIBridge PCIBridge;
typedef struct PCIBus PCIBus;
typedef struct PCIDevice PCIDevice;
typedef struct PCIEAERErr PCIEAERErr;
typedef struct PCIEAERLog PCIEAERLog;
typedef struct PCIEAERMsg PCIEAERMsg;
typedef struct PCIEPort PCIEPort;
typedef struct PCIESlot PCIESlot;
typedef struct PCIExpressDevice PCIExpressDevice;
typedef struct PCIExpressHost PCIExpressHost;
typedef struct PCIHostDeviceAddress PCIHostDeviceAddress;
typedef struct PCIHostState PCIHostState;
typedef struct PCMachineState PCMachineState;
typedef struct PostcopyDiscardState PostcopyDiscardState;
typedef struct Property Property;
typedef struct PropertyInfo PropertyInfo;
typedef struct QBool QBool;
typedef struct QDict QDict;
typedef struct QEMUBH QEMUBH;
typedef struct QemuConsole QemuConsole;
typedef struct QEMUFile QEMUFile;
typedef struct QemuLockable QemuLockable;
typedef struct QemuMutex QemuMutex;
typedef struct QemuOpt QemuOpt;
typedef struct QemuOpts QemuOpts;
typedef struct QemuOptsList QemuOptsList;
typedef struct QEMUSGList QEMUSGList;
typedef struct QemuSpin QemuSpin;
typedef struct QEMUTimer QEMUTimer;
typedef struct QEMUTimerListGroup QEMUTimerListGroup;
typedef struct QJSON QJSON;
typedef struct QList QList;
typedef struct QNull QNull;
typedef struct QNum QNum;
typedef struct QObject QObject;
typedef struct QString QString;
typedef struct RAMBlock RAMBlock;
typedef struct Range Range;
typedef struct SHPCDevice SHPCDevice;
typedef struct SSIBus SSIBus;
typedef struct VirtIODevice VirtIODevice;
typedef struct Visitor Visitor;
typedef void SaveStateHandler(QEMUFile *f, void *opaque);
typedef int LoadStateHandler(QEMUFile *f, void *opaque, int version_id);

#endif /* QEMU_TYPEDEFS_H */
```

As you probably already noted, since Qemu uses C, it has to do some tricks to emulate the power of object oriented languages like C++. Therefore, it has lots of helper macros that can extract parent classes from other classes and etc. It's called Qemu Object Model, and it's defined and explained in [/qom/object.h](). The file is too big to post here, but there you'll find this little introduction:

 * The QEMU Object Model provides a framework for registering user creatable
 * types and instantiating objects from those types.  QOM provides the following
 * features:
 *
 *  - System for dynamically registering types
 *  - Support for single-inheritance of types
 *  - Multiple inheritance of stateless interfaces
 *

The central struct in all of this is the `TypeInfo` struct. It models how the instance of the class is going to be instantiated, by explaining how the parents should be instantiated. It also does some things like defining the interfaces it adheres, and if the class is abstract.

Let's take a closer look at TypeInfo, at [object.h]():

```C
/**
 * TypeInfo:
 * @name: The name of the type.
 * @parent: The name of the parent type.
 * @instance_size: The size of the object (derivative of #Object).  If
 *   @instance_size is 0, then the size of the object will be the size of the
 *   parent object.
 * @instance_init: This function is called to initialize an object.  The parent
 *   class will have already been initialized so the type is only responsible
 *   for initializing its own members.
 * @instance_post_init: This function is called to finish initialization of
 *   an object, after all @instance_init functions were called.
 * @instance_finalize: This function is called during object destruction.  This
 *   is called before the parent @instance_finalize function has been called.
 *   An object should only free the members that are unique to its type in this
 *   function.
 * @abstract: If this field is true, then the class is considered abstract and
 *   cannot be directly instantiated.
 * @class_size: The size of the class object (derivative of #ObjectClass)
 *   for this object.  If @class_size is 0, then the size of the class will be
 *   assumed to be the size of the parent class.  This allows a type to avoid
 *   implementing an explicit class type if they are not adding additional
 *   virtual functions.
 * @class_init: This function is called after all parent class initialization
 *   has occurred to allow a class to set its default virtual method pointers.
 *   This is also the function to use to override virtual methods from a parent
 *   class.
 * @class_base_init: This function is called for all base classes after all
 *   parent class initialization has occurred, but before the class itself
 *   is initialized.  This is the function to use to undo the effects of
 *   memcpy from the parent class to the descendants.
 * @class_data: Data to pass to the @class_init,
 *   @class_base_init. This can be useful when building dynamic
 *   classes.
 * @interfaces: The list of interfaces associated with this type.  This
 *   should point to a static array that's terminated with a zero filled
 *   element.
 */
struct TypeInfo
{
    const char *name;// name of the type
    const char *parent;// name of the parent type

    size_t instance_size;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;
    size_t class_size;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void *class_data;

    InterfaceInfo *interfaces;
};
```

You can see a lot of concepts from object oriented languages are applied here. For now, let's establish the difference between `instance_something` and `class_something`. For example, `instance_init` does exactly what you'd expect: it initializes data members for the instance, like a constructor. The `class_init`, on the other hand, creates the object class. Since C is low level, if we want to implement OOP concepts, we need to do what these languages do for us in the background: set the virtual method function pointers as well as manipulate the class 'methods' and override the ones that were overwritten by the 'lower class'. 

Here's a real world example from [/hw/block/nvme.c](), with the `pci_device_type_info` borrowed from [/hw/pci/pci.c]():

```C
static const TypeInfo pci_device_type_info = {
    .name = TYPE_PCI_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(PCIDevice),
    .abstract = true,
    .class_size = sizeof(PCIDeviceClass),
    .class_init = pci_device_class_init,
    .class_base_init = pci_device_class_base_init,
};

static void nvme_class_init(ObjectClass *oc, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(oc);
    PCIDeviceClass *pc = PCI_DEVICE_CLASS(oc);

    pc->realize = nvme_realize;
    pc->exit = nvme_exit;
    pc->class_id = PCI_CLASS_STORAGE_EXPRESS;
    pc->vendor_id = PCI_VENDOR_ID_INTEL;
    pc->device_id = 0x5845;
    pc->revision = 2;

    set_bit(DEVICE_CATEGORY_STORAGE, dc->categories);
    dc->desc = "Non-Volatile Memory Express";
    dc->props = nvme_props;
    dc->vmsd = &nvme_vmstate;
}

static void nvme_instance_init(Object *obj)
{
    NvmeCtrl *s = NVME(obj);

    device_add_bootindex_property(obj, &s->conf.bootindex,
                                  "bootindex", "/namespace@1,0",
                                  DEVICE(obj), &error_abort);
}

static const TypeInfo nvme_info = {
    .name          = TYPE_NVME,
    .parent        = TYPE_PCI_DEVICE,
    .instance_size = sizeof(NvmeCtrl),
    .class_init    = nvme_class_init,
    .instance_init = nvme_instance_init,
    .interfaces = (InterfaceInfo[]) {
        { INTERFACE_PCIE_DEVICE },
        { }
    },
};
```

How does Qemu creates an instance for the nvme class? Well, it first traverses through the parents from `.parent` for each type and initialize each of the classes: sets the virtual method pointers and possibly overwirtes some of them. You can see by the code above that the NVME class has PCI_DEVICE class as parent, which has a DEVICE class as its parent, and so on. 

Then, Qemu initialize things by calling the `.instance_init` functions from each of the classes. It traverses the parents the same way and initializes their members first, then the derived members.

Take a look at `nvme_class_init`: you can see that it gets the class for the parent, which is `PCIDeviceClass`. There, it overwrites the `realize` 'method' (in C++ you don't call things methods but simply functions). Now you know how to method override works.

You can also see that it has, as interface, the `INTERFACE_PCIE_DEVICE`, which is an interface that every PCIE device must implement. You understand now how polymorphism is implemented into the C language.

We'll talk more about how to initialize objects using the functions and macros defined in [/qom/object.h](), but for now it suffices to understand how classes are related.