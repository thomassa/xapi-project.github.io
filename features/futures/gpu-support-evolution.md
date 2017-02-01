---
title: GPU support evolution
layout: default
design_doc: true
revision: 4
status: draft
revision_history:
- revision_number: 1
  description: Documented interface changes between xapi and xenopsd for vGPU
- revision_number: 2
  description: Added design for storing vGPU-to-pGPU allocation in xapi database
- revision_number: 3
  description: Marked new xapi DB fields as internal-only
- revision_number: 4
  description: Extended design to include the intentions for AMD MxGPU
---

Introduction
------------

This should be read in conjunction with
({{site.baseurl}}/features/VGPU/vgpu.html) and
({{site.baseurl}}/xapi/futures/vgpu-type-identifiers.html)

As of XenServer 6.5, VMs can be provisioned with access to graphics processors
(either emulated or passed through) in four different ways.

In XenServer 7.0 and later, virtualisation of Intel graphics processors exists
as a fifth kind of graphics processing available to VMs.

Work is in progress towards support for AMD MxGPU ("multi-user GPU") as a
sixth kind.

These six situations all require the VM's device model to be created
in subtly different ways:

__Pure software emulation__

- qemu is launched either with no special parameter, if the basic Cirrus
  graphics processor is required, otherwise qemu is launched with the
  `-std-vga` flag.

__Generic GPU passthrough__

- qemu is launched with the `-priv` flag to turn on privilege separation
- qemu can additionally be passed the `-std-vga` flag to choose the
  corresponding emulated graphics card.

__Intel integrated GPU passthrough (GVT-d)__

- As well as the `-priv` flag, qemu must be launched with the `-std-vga` and
  `-gfx_passthru` flags. The actual PCI passthrough is handled separately
  via xen.

__NVIDIA vGPU__

- qemu is launched with the `-vgpu` flag
- a secondary display emulator, demu, is launched with the following parameters:
  - `--domain` - the VM's domain ID
  - `--vcpus` - the number of vcpus available to the VM
  - `--gpu` - the PCI address of the physical GPU on which the emulated GPU will
    run
  - `--config` - the path to the config file which contains detail of the GPU to
      emulate

__Intel vGPU (GVT-g)__

- here demu is not used, but instead qemu is launched with five parameters:
  - `-xengt`
  - `-vgt_low_gm_sz` - the low GM size in MiB
  - `-vgt_high_gm_sz` - the high GM size in MiB
  - `-vgt_fence_sz` - the number of fence registers
  - `-priv`

__AMD vGPU (MxGPU)__

(To be implemented)

A VF (one of the GPU's virtual functions) is treated as a PCI device, and is
made available to the VM through the PCI passthrough mechanism: the pci device
is specified via xenstore as a hotplug event (pci_ins format).

- here (as for Intel GVT-g) demu is not used, but instead qemu is launched with
  the following parameters in order to configure the VF:
  - `-sched` - the scheduling slice
  - `-fbsize` - the framebuffer size in bytes

__

xenopsd
-------

To handle all these possibilities, we added some types to xenopsd's
interface:

```
module Pci = struct
  type address = {
    domain: int;
    bus: int;
    dev: int;  (* device *)
    fn: int;
  }

  ...
end

module Vgpu = struct
  type gvt_g = {
    physical_pci_address: Pci.address;
    low_gm_sz: int64;
    high_gm_sz: int64;
    fence_sz: int;
  }

  type nvidia = {
    physical_pci_address: Pci.address;
    config_file: string
  }

  type mxgpu = {
    physical_pci_address: Pci.address; (* of Virtual Function *)
    sched: int;
    fb_size: int64;
  }

  type implementation =
    | GVT_g of gvt_g
    | Nvidia of nvidia
    | MxGPU of mxgpu

  type id = string * string

  type t = {
    id: id;
    position: int;
    implementation: implementation;
  }

  type state = {
    plugged: bool;
    emulator_pid: int option;
  }
end

module Vm = struct
  type igd_passthrough of
    | GVT_d

  type video_card =
    | Cirrus
    | Standard_VGA
    | Vgpu
    | Igd_passthrough of igd_passthrough

  ...
end

module Metadata = struct
  type t = {
    vm: Vm.t;
    vbds: Vbd.t list;
    vifs: Vif.t list;
    pcis: Pci.t list;
    vgpus: Vgpu.t list;
    domains: string option;
  }
end
```

The `video_card` type is used to indicate to the function
`Xenops_server_xen.VM.create_device_model_config` how the VM's emulated graphics
card will be implemented. A value of `Vgpu` indicates that the VM needs to be
started with one or more virtualised GPUs - the function will need to look at
the list of GPUs associated with the VM to work out exactly what parameters to
send to qemu.

If `Vgpu.state.emulator_pid` of a plugged vGPU is `None`, this indicates that
the emulation of the vGPU is being done by qemu rather than by a separate
emulator.

n.b. adding the `vgpus` field to `Metadata.t` will break backwards compatibility
with old versions of xenopsd, so some upgrade logic will be required.

This interface will allow us to support multiple vGPUs per VM in future if
necessary, although this may also require reworking the interface between
xenopsd, qemu and demu. For now, xenopsd will throw an exception if it is asked
to start a VM with more than one vGPU.

xapi
----

To support the above interface, xapi will convert all of a VM's non-passthrough
GPUs into `Vgpu.t` objects when sending VM metadata to xenopsd.

#### Intel GVT-g
In contrast to GVT-d, which can only be run on an Intel GPU which has been
hidden from dom0, GVT-g will only be allowed to run on a GPU which has _not_
been hidden from dom0.

If a GVT-g-capable GPU is detected, and it is not hidden from dom0, xapi will
create a set of VGPU_type objects to represent the vGPU presets which can run on
the physical GPU. These presets are defined through a "whitelist" config file,
where each line in the file represents a pairing of PCI device-id and a
specification for a VGPU_type. Each line is scanned using the format-string

`"%04x experimental=%c name='%s@' low_gm_sz=%Ld high_gm_sz=%Ld fence_sz=%Ld framebuffer_sz=%Ld max_heads=%Ld resolution=%Ldx%Ld monitor_config_file=%s"`

An example from a test-file is

`1234 experimental=0 name='GVT-g on 1234' low_gm_sz=128 high_gm_sz=384 fence_sz=4 framebuffer_sz=128 max_heads=1 resolution=1920x1080 monitor_config_file=/path/to/file`

#### AMD MxGPU

This type of GPU requres a driver (a kernel module) to be loaded in dom0 if
its virtualisation functionality is to be used.
Once this module is in use, additional devices can be seen on the PCI bus.
These are the VFs (virtual functions) provided by the GPU. The original PCI
device ID remains present; this corresponds to the PF (physical function).
On the PCI bus (tree), the VFs appear as children of their PF device.

To reflect this, in the xapi datamodel the pci class needs a pair of fields
(with default values of empty-list and null-ref respectively)
- virtual_functions: the pci objects of the VFs of this PF pci device
- physical_function: the pci object of the PF that hosts this VF pci device

The first step is similar to Intel GVT-g: if the host's PCI devices include
an MxGPU PF, then xapi will create a set of VGPU_type objects to represent the
vGPU presets which can run on the physical GPU. These presets will be defined
in essentially the same way as for Intel GVT-g, by means of a "whitelist"
config file in which xapi use the PCI device id to look up the presets
(VGPU_type specifications). The set of fields is:
- `pdev_id` (id of the PF)
- `name` (human-readable)
- `fb_size` (framebuffer size)
- `sched` (scheduling slice)
- `max_per_pgpu` (a pGPU can run this many vGPUs of this VGPU_type)

### Allocation of vGPUs to physical GPUs

#### NVIDIA vGPU
For NVIDIA vGPU, when starting a VM, each vGPU attached to the VM is assigned
to a physical GPU as a result of capacity planning at the pool level. The
resulting configuration is stored in the VM.platform dictionary, under
specific keys:

- `vgpu_pci_id` - the address of the physical GPU on which the vGPU will run
- `vgpu_config` - the path to the vGPU config file which the emulator will use

Instead of storing the assignment in these fields, we will add a new
internal-only database field:

- `VGPU.scheduled_to_be_resident_on (API.ref_PGPU)`

This will be set to the ref of the physical GPU on which the vGPU will run. From
here, xapi can easily obtain the GPU's PCI address. Capacity planning will also
take into account which vGPUs are scheduled to be resident on a physical GPU,
which will avoid races resulting from many vGPU-enabled VMs being started at
once.

The path to the config file is already stored in the `VGPU_type.internal_config`
dictionary, under the key `vgpu_config`. xapi will use this value directly
rather than copying it to VM.platform.

To support other vGPU implementations, we have another internal-only
database field:

- `VGPU_type.implementation enum(Passthrough|Nvidia|GVT_g|MxGPU)`

#### Intel GVT_g
For the `GVT_g` implementation, no config file is needed. Instead,
`VGPU_type.internal_config` will contain three key-value pairs, with the keys

- `vgt_low_gm_sz`
- `vgt_high_gm_sz`
- `vgt_fence_sz`

The values of these pairs will be used to construct a value of type
`Xenops_interface.Vgpu.gvt_g`, which will be passed down to xenopsd.

#### AMD MxGPU
When starting a VM using a vGPU of this category, if the `gim.ko` kernel module
has not yet been loaded on the chosen host, then xapi must:
- load the `gim.ko` kernel module on that host, then
- create database entries for the new PCI devices that appear (`Xapi_pci.update_pcis`)

These new PCI devices that correspond to VFs (and thus have non-null
`physical_function` fields) are for internal use only and must be barred
from explicit conventional pass-through by the user. There will be no
pgpu objects corresponding to the VF pci objects: only the PF pci has a
corresponding pgpu.

Once the kernel module is loaded, xapi must:
- assign each virtual GPU to a physical GPU (as in the NVidia case)
- assign each virtual GPU to a Virtual Function PCI device on that pGPU.

`VGPU.scheduled_to_be_resident_on (API.ref_PGPU)` will be set to the VF,
to be passed through as in generic passthrough.
From the VF ref_PGPU, xapi can easily obtain the VF's PCI address and also the
address of the parent PF; the latter will be used for capacity planning.

The VGPU_type object will contain the `sched` and `fb_size` values that
are used (along with the VF's PCI address) to construct a value of type
`Xenops_interface.Vgpu.mxgpu`, which will be passed down to xenopsd.
