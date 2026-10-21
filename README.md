# DroidDeck GPU Fix

[中文说明](README.zh-CN.md)

A KernelSU module that restores GPU acceleration for DroidDeck
containers on HyperOS port ROMs.

## Problem

On a HyperOS 4 port ROM (Lenovo Legion Y700 4th gen, Snapdragon 8
Elite / Adreno 830), DroidDeck containers silently run on llvmpipe:
GPU 0%, CPU ~53%, games jittering around 40 fps. The same container
setup on a ColorOS port ROM uses the GPU normally. The GPU device
node `/dev/kgsl-3d0` exists, is world-readable (0666) and carries the
correct SELinux label (`gpu_device`). Setting SELinux permissive
revives the GPU instantly (120 fps, GPU 31%) — so the policy, not
the driver or the node, is at fault.

## Root cause

Captured AVC denials (permissive mode) show **zero** denials against
the GPU device node itself. What is denied — silently, via dontaudit,
in enforcing mode — is the app domain's (`untrusted_app_27`) GPU
*identity probing* at session start:

- read/open/getattr on `vendor_sysfs_kgsl` files (model, temp,
  gpu_busy_percentage)
- open/getattr/map on property contexts `vendor_soc_model_prop` and
  `vendor_default_prop`
- search on `sysfs` directories (hwmon)

DroidDeck's session-start helper reads those sysfs files and
properties to identify the GPU. Denied, it concludes there is no GPU
and starts the session with software rendering — it never even opens
the device node. (Half of the "GPU 0%" reading is the overlay's own
telemetry reads being denied, too.)

The port's merged SELinux policy types the donated vendor files as
vendor-private and never grants the app domain access. Same porter,
different donor ROM, different gap: the ColorOS port's policy does
not cut this path.

## Fix

`sepolicy.rule` adds four narrow allow rules instead of running the
device permissive:

```
allow untrusted_app_27 vendor_sysfs_kgsl file { read open getattr }
allow untrusted_app_27 vendor_soc_model_prop file { read open getattr map }
allow untrusted_app_27 vendor_default_prop file { read open getattr map }
allow untrusted_app_27 sysfs dir { search }
```

## Install

Flash the module in SukiSU / KernelSU and **fully reboot** — a soft
reboot is not enough, sepolicy rules load at boot. Verify with
`getenforce` (must be `Enforcing`) and the DroidDeck overlay
(GPU% alive, no longer pinned at 0).

## Notes

- If a DroidDeck update changes its SELinux domain (currently
  `untrusted_app_27`), update the domain name in `sepolicy.rule`.
- Deliberately not granted in v1.0: execstack, proc/debugfs telemetry
  — not needed for GPU detection on the verified setup.

## Author & license

Author: Muse · MIT License
