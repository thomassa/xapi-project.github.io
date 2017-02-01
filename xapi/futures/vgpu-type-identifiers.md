---
title: VGPU type identifiers
layout: default
design_doc: true
revision: 2
status: draft
design_review: 157
revision_history:
- revision_number: 2
  description: addition of AMD MxGPU
- revision_number: 1
  description: Initial version
  released (7.0)
---

Introduction
------------

When xapi starts, it may create a number of VGPU_type objects. These act as
VGPU presets, and exactly which VGPU_type objects are created depends on the
installed hardware and in certain cases the presence of certain files in dom0.

When deciding which VGPU_type objects need to be created, xapi needs to
determine whether a suitable VGPU_type object already exists, as there should
never be duplicates. Originally the combination of vendor name and model name
was used as a primary key, but this is not ideal as these values are subject to
change. We therefore need a way of creating a primary key to uniquely identify
VGPU_type objects.

Identifier
----------

We added a new read-only field to the database:

- `VGPU_type.identifier (string)`

This field contains a string representation of the parameters required to
uniquely identify a VGPU_type. The parameters required can be summed up with the
following OCaml type:

```
type nvidia_id = {
  pdev_id : int;
  psubdev_id : int option;
  vdev_id : int;
  vsubdev_id : int;
}

type gvt_g_id = {
  pdev_id : int;
  low_gm_sz : int64;
  high_gm_sz : int64;
  fence_sz : int64;
  monitor_config_file : string option;
}

type mxgpu_id = {
  pdev_id : int; (* PCI id of the PF, not the VFs *)
  sched: int;
  fb_size: int64;
}

type t =
  | Passthrough
  | Nvidia of nvidia_id
  | GVT_g of gvt_g_id
  | MxGPU of mxgpu_id
```

When converting this type to a string, the string will always be prefixed with
`0001:` enabling future versioning of the serialisation format.

For passthrough, the string will simply be:

`0001:passthrough`

For NVIDIA, the string will be `nvidia` followed by the four device IDs
serialised as four-digit hex values, separated by commas. If `psubdev_id` is
`None`, the empty string will be used e.g.

```
Nvidia {
  pdev_id = 0x11bf;
  psubdev_id = None;
  vdev_id = 0x11b0;
  vsubdev_id = 0x109d;
}
```

would map to

`0001:nvidia,11bf,,11b0,109d`

For GVT-g, the string will be `gvt-g` followed by the physical device ID encoded
as four-digit hex, followed by `low_gm_sz`, `high_gm_sz` and `fence_sz` encoded
as hex, followed by `monitor_config_file` (or the empty string if it is `None`)
e.g.

```
GVT_g {
  pdev_id = 0x162a;
  low_gm_sz = 128L;
  high_gm_sz = 384L;
  fence_sz = 4L;
  monitor_config_file = None;
}
```

would map to

`0001:gvt-g,162a,80,180,4,,`

For AMD MxGPU, the string will be, in order:
* `mxgpu`
* the physical devicde ID encoded as four-digit hex
* the scheduling slice encoded as hex
* the framebuffer size in bytes, encoded as hex

e.g.
```
MxGPU {
  pdev_id = 0xa987
  sched = 3
  fb_size = 8388608L
}
```
would map to

`0001:mxgpu,a987,3,800000`

Having this string in the database will allow us to do a simple lookup to test
whether a certain VGPU_type already exists. Although it is not currently
required, this string can also be converted back to the type from which it was
generated.

When deciding whether to create VGPU_type objects, xapi will generate the
identifier string and use it to look for existing VGPU_type objects in the
database. If none are found, xapi will look for existing VGPU_type objects with
the tuple of model name and vendor name. If still none are found, xapi will
create a new VGPU_type object.

Note that in every case the second field of the string stored in
`VGPU_type.identifier` corresponds to the enum value stored in
`VGPU_type.implementation (vgpu_type_implementation)` of that VGPU_type.
The enum contains "passthrough", "nvidia", "gvt_g" and "mxgpu".
