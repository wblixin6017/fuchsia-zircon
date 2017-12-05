# Zircon Driver Development

Zircon drivers are shared libraries that are dynamically loaded in Device Host
processes in user space. The process of loading a driver is controlled by the
Device Coordinator. See [Device Model](device-model.md) for more information on
Device Hosts, Device Coordinator and the driver and device lifecycles.

## Directory structure

Zircon drivers are found under [system/dev](../../system/dev). They are
grouped by device class. In the driver's `rules.mk`, the `MODULE_TYPE` should
be `driver`. This will install the driver shared lib in `/boot/driver/`.

If your driver is built outside Zircon, install them in `/system/driver/`. The
Device Coordinator looks in those directories for loadable drivers.

## Declaring a driver

At a minimum, a driver should contain the driver declaration and implement the
`bind()` driver op.

Drivers are loaded and bound to a device when the devcoordinator
successfully finds a matching driver for a device. A driver declares the
devices it is compatible with via bindings. The following bind program
declares the [AHCI driver](../../system/dev/block/ahci/ahci.c):

```
ZIRCON_DRIVER_BEGIN(ahci, ahci_driver_ops, "zircon", "0.1", 4)
    BI_ABORT_IF(NE, BIND_PROTOCOL, ZX_PROTOCOL_PCI),
    BI_ABORT_IF(NE, BIND_PCI_CLASS, 0x01),
    BI_ABORT_IF(NE, BIND_PCI_SUBCLASS, 0x06),
    BI_MATCH_IF(EQ, BIND_PCI_INTERFACE, 0x01),
ZIRCON_DRIVER_END(ahci)
```

The AHCI driver has 4 directives in the bind program. `"zircon"` is the vendor
id and `"0.1"` is the driver version. It binds with `ZX_PROTOCOL_PCI` devices
with PCI class 1, subclass 6, interface 1.

The [PCI driver](../../system/dev/bus/pci/kpci.c) publishes the matching
device with the following properties:

```
zx_device_prop_t device_props[] = {
    {BIND_PROTOCOL, 0, ZX_PROTOCOL_PCI},
    {BIND_PCI_VID, 0, info.vendor_id},
    {BIND_PCI_DID, 0, info.device_id},
    {BIND_PCI_CLASS, 0, info.base_class},
    {BIND_PCI_SUBCLASS, 0, info.sub_class},
    {BIND_PCI_INTERFACE, 0, info.program_interface},
    {BIND_PCI_REVISION, 0, info.revision_id},
    {BIND_PCI_BDF_ADDR, 0, BIND_PCI_BDF_PACK(info.bus_id, info.dev_id,
                                             info.func_id)},
};
```

Binding variables and macros are defined in
[zircon/driver/binding.h](../../system/public/zircon/driver/binding.h).

If you are introducing a new device class, you may need to introduce new
binding variables in that file. Binding variables are 32-bit values. If your
variable value is bigger, split them into multiple 32-bit variables. An
example is ACPI HID values, which are 8 characters (64-bits) long. We define
`BIND_ACPI_HID_0_3` and `BIND_ACPI_HID_4_7`.

Binding directives are evaluated sequentially. The branching directives
`BI_GOTO()` and `BI_GOTO_IF()` allows you to jump forward to the matching
label, defined by `BI_LABEL()`.

`BI_ABORT_IF_AUTOBIND` means the driver will not be considered in automatic
device matching. It is only bindable manually. To bind a driver to a device,
call `ioctl_device_bind()` on the device to bind.

Device classes are defined in
[ddk/include/ddk/protodefs.h](../../system/ulib/ddk/include/ddk/protodefs.h).

## Driver binding

A driver’s `bind()` function is called when it is matched to a device.
Generally a driver will initialize any data structures needed for the device
and initialize hardware in this function. It should not perform any
time-consuming tasks or block in this function, because it is invoked from the
devhost’s RPC thread and it will not be able to service other requests in the
meantime. Instead, it should spawn a new thread to perform lengthy tasks.

The driver should not make assumptions about the state of the hardware in
`bind()` and should reset the hardware. Because the system recovers from a
driver crash by re-spawning the devhost, the hardware may be in an unknown
state when `bind()` is invoked.

A driver is required to publish a `zx_device_t` in `bind()` by calling
`device_add()`. This is necessary for the devcoordinator to keep track of the
device lifecycle. If the driver is not able to publish a functional device in
`bind()`, for example if it is initializing the full device in a thread, it
should publish an invisible device, and make this device visible when
initialization is completed. See `DEVICE_ADD_INVISIBLE` and
`device_make_visible()` in
[zircon/ddk/driver.h](../../system/ulib/ddk/include/ddk/driver.h).

There are generally four outcomes from `bind()`:

1. The driver determines that even though the bind program matched, the device
cannot be supported (maybe due to checking hw version bits or whatnot) and
returns an error.

2. The driver determines the device is supported and does not need to do any
heavy lifting, so publishes a new device via `device_add()` and returns
`ZX_OK`.

3. The driver needs to do further initialization before the device is ready or
it’s sure it can support it, so it publishes an invisible device and kicks off
a thread to keep working, while returning `ZX_OK`. That thread will eventually
make the device visible or, if it cannot successfully initialize it, remove it.

4. The driver represents a bus or controller with 0..n children which may
dynamically appear or disappear. In this case it should publish a device
immediately representing the bus or controller, and then dynamically publish
children (that downstream drivers will bind to) representing hardware on that
bus. Examples: AHCI/SATA, USB, etc.

After a device is added and made visible by the system, it is made available
to client processes and for binding by compatible drivers.

## Device protocols

A driver provides a set of device ops and optional protocol ops to a device.
Device ops implement the device lifecycle methods and the external interface
to the device that are called by other user space applications and services.
Protocol ops implement the ddk-internal protocols of the device that are
called by other drivers.

You can pass one set of protocol ops for the device in `device_add_args_t`. If
a device supports multiple protocols, implement the `get_protocol()` device
op. A device can only have one protocol id. The protocol id corresponds to the
class the device is published under in devfs.

Device protocol headers are found in
[ddk/protocol/](../../system/ulib/ddk/include/ddk/protocol). Ops and any data
structures passed between drivers should be defined in this header.

## Driver operation

A driver generally operates by servicing client requests from children drivers
or other processes. It fulfills those requests either by communicating
directly with hardware (for example, via MMIO) or by communicating with its
parent device (for example, queuing a USB transaction).

External client requests from processes outside the devhost are fulfilled by
the device ops `read()`, `write()`, `ioctl()` and `iotxn_queue()`. Requests
from children drivers, generally in the same process, are fulfilled by device
protocols corresponding to the device class. With the exception of
`iotxn_queue()`, it is preferable to use device protocols between drivers
instead of device ops.

A device can get a protocol supported by its parent by calling
`device_get_protocol()` on its parent device.

The `iotxn_queue()` device op is optional. It is only implemented by drivers
that support asynchronous transactions. If a driver implements
`iotxn_queue()`, it does not need to implement `read()` and `write()`.
Sync operations will be submitted as iotxns.

## Iotxns

[Iotxns](../../system/ulib/ddk/include/ddk/iotxn.h) is a mechanism to
implement asynchronous transactions. New transactions are submitted to the
driver in `iotxn_queue()`. The `completion()` callback in the iotxn is invoked
when the driver finishes processing the iotxn. A driver may handle the iotxn
itself, or pass it to its parent.

All iotxns are backed by VMOs. `iotxn_clone()` points the cloned iotxn to the
same VMO, and does not copy data. It is possible to clone an iotxn to a
subrange using `iotxn_clone_partial()`. This is useful when the client
passes in a transaction bigger than what the hardware can support.

It is valid for an intermediate processor of the iotxn to modify the offset
and length to a subrange. For example, the GPT partition driver sets the
offset of the incoming iotxn to be relative to the start of the parent device.
Because of this, a driver should treat iotxn objects as opaque in the
`completion()` callback and only use the `status` and `actual` arguments
passed to the callback.

Each iotxn provides 48 bytes of storage for protocol-specific data in
`protocol_data` and a field defining the protocol of the data in `protocol`.
For example, the [../system/dev/block/sdmmc/sdmmc.c](SD/MMC driver) stores the
SD/MMC command and argument to issue to the controller in this field.

An iotxn caches the physical mapping of the VMO once it is mapped. If you
re-use a VMO, the system will not need to remap the physical pages every time
it is processed. If the physical mapping is already known, it is also possible
to pass in the physical mapping directly by setting the `phys` and
`phys_count` fields.

`iotxn_phys_iter_t` is a convenient way to create scatter-gather tables out of
iotxns. `iotxn_phys_iter_next()` returns chunks of contiguous ranges of a
maximum length for each entry of the scatter-gather table.

## Device interrupts

Device interrupts are implemented by interrupt objects, a kernel object. A
driver requests a handle to the device interrupt from its parent device in a
device protocol method. For example, the PCI protocol implements
`map_interrupt()` for PCI children. A driver should spawn a thread to wait on
the interrupt handle, which will be signaled when the system receives an
interrupt. The interrupt is masked when the object is signaled. Call
[zx_interrupt_complete()](../syscalls/interrupt_wait.md) to unmask the
interrupt.

The interrupt thread should not perform any long-running tasks. For drivers
that perform lengthy tasks, use a worker thread.

You can signal an interrupt handle with
[zx_interrupt_signal()](../syscalls/interrupt_signal.md) to return from
`zx_interrupt_wait()`. This is necessary to shut down the interrupt thread
during driver clean up.

## ioctls

Ioctls for each device class are defined in
[zircon/device/](../../system/public/zircon/device). Ioctls may accept or
return handles. The `IOCTL_KIND_*` defines in
[zircon/device/ioctl.h](../../system/public/zircon/device/ioctl.h), used in
the ioctl declaration, defines whether the ioctl accepts or returns handles
and how many. The driver owns the handles passed in and should close the
handles when they’re no longer needed, unless it returns
`ZX_ERR_NOT_SUPPORTED` in which case the devhost RPC layer will close the
handles.

## Protocol ops vs. ioctls

Protocol ops define the DDK-internal API for a device. Ioctls define the
external API. Define a protocol op if the function is primarily meant to be
called by other drivers, and generally a driver should call a protocol op on
its parent instead of an ioctl.

## Isolate devices

Devices that are added with `DEVICE_ADD_MUST_ISOLATE` spawn a new devhost. The
device exists in both the parent devhost and as the root of the new devhost.
The driver is provided a channel in `create()` when it creates the proxy
device, or the “bottom half” that runs in the new devhost. The proxy device
should cache this channel for when it needs to communicate with the top half,
for example if it needs to call API on the parent device.

`rxrpc()` is invoked on the top half when this channel is written to by the
bottom half. There is no common wire protocol for this channel. For an
example, refer to the [PCI driver](../../system/dev/bus/pci).

NOTE: This is a mechanism used by various bus devices and not something
general drivers should have to worry about. (please ping swetland if you think
you need to use this)

## Logging

[ddk/debug.h](../../system/ulib/ddk/include/ddk/debug.h) defines the
`zxlogf(<log_level>,...)` macro. The log messages are printed to the system
debuglog over the network and on the serial port if available for the device.
By default, `ERROR` and `INFO` are always printed. You can control the log
level for a driver by passing the boot cmdline
`driver.<driver_name>.log=+<level>,-<level>`. For example,
`driver.sdhci.log=-info,+trace,+spew` enables the `TRACE` and `SPEW` logs and
disable the `INFO` logs for the sdhci driver.

The log levels prefixed by "L" (`LERROR`, `LINFO`, etc.) do not get sent over
the network and is useful for network logging.

## Driver testing

`ZX_PROTOCOL_TEST` provides a mechanism to test drivers by running the driver
under test in an emulated environment. Write a driver that binds to a
`ZX_PROTOCOL_TEST` device. This driver should publish a device that the driver
under test can bind to, and it should implement the protocol functions the
driver under test invokes in normal operation. This helper driver should be
declared with `BI_ABORT_IF_AUTOBIND` in the bindings.

The test harness calls `ioctl_test_create_device()` on `/dev/misc/test`, which
will create a `ZX_PROTOCOL_TEST` device and return its path. Then it calls
`ioctl_device_bind()` with the helper driver on the newly created device.
This approach generally works better for mid-layer protocol drivers. It's
possible to emulate real hardware with the same approach but it may not be as
useful.

The functions defined in
[ddk/protocol/test.h](../../system/ulib/ddk/include/ddk/protocol/test.h) are
for testing libraries that run as part of a driver. For an example, refer to
[system/ulib/ddk/test](../../system/ulib/ddk/test). The test harness for these
tests is
[system/utest/driver-tests/main.c](../../system/utest/driver-tests/main.c)

## Driver rights

Although drivers run in user space processes, they have a more restricted set
of rights than normal processes. Drivers are not allowed to access the
filesystem, including devfs. That means a driver cannot interact with
arbitrary devices. If your driver needs to do this, consider writing a service
instead. For example, the virtual console is implemented by the
[virtcon](../../system/core/virtcon) service.

Privileged operations such as `zx_vmo_create_contiguous()` and
[zx_interrupt_create](../syscalls/interrupt_create.md) require a root resource
handle. This handle is not available to drivers other than the system driver
([ACPI](../../system/dev/bus/acpi) on x86 systems and
[platform](../../system/dev/bus/platform) on ARM systems). A device should
request its parent to perform such operations for it. Contact the author
of the parent driver if its protocol does not address this use case.

Similarly, a driver is not allowed to request arbitrary MMIO ranges,
interrupts or GPIOs. Bus drivers such as PCI and platform only return the
resources associated to the child device.