# POCO F7 OxygenOS 50MP camera fix

A compiled ARM64 preload shim that fixes half/partial Ultra HD photos on the
tested POCO F7 OxygenOS port. Original `libcameraservice.so` and cameraserver
binaries stay intact. This repository publishes compiled binaries and partition
integration files; the implementation source is maintained privately.

## Downloads

Get the compiled library or partition bundle from
[Releases](https://github.com/nischay12089/poco-f7-50mp-fix/releases).

- `libcamera50mp_shim.so`: tested preload library.
- `camera50mp-partition-v1.0.0-arm64.zip`: library, init configuration, metadata,
  and installation instructions for integrating into an extracted system image.
- `SHA256SUMS.txt`: release asset hashes.

The ZIP is for **ROM partition integration**. It is not a recovery-flashable
ZIP and does not install a root module.

## Tested compatibility

| Item | Tested value |
| --- | --- |
| Hardware | POCO F7 / onyx / ARM64 |
| ROM | OxygenOS port `CPH2573_16.0.10.501(EX01)` |
| Android | 16 |
| Camera app | `6.6.000460.0_ireddragonicy` |
| SELinux during testing | Permissive |

The release targets this camera-service ABI. Compatibility with another ROM,
camera library version, or enforcing SELinux has not been validated.
The original library/executable hashes are recorded in [manifest.json](manifest.json).

## Installation into a system image

Copy the supplied files into the extracted system partition:

| Device path | Owner | Mode | SELinux context |
| --- | --- | --- | --- |
| `/system/lib64/libcamera50mp_shim.so` | 0:0 | 0644 | `u:object_r:system_lib_file:s0` |
| `/system/etc/init/cameraserver.rc` | 0:0 | 0644 | `u:object_r:system_file:s0` |

The supplied RC comes from the tested ROM and adds this service environment
setting, supported by [Android init](https://android.googlesource.com/platform/system/core/+/refs/heads/main/init/README.md):

```rc
service cameraserver /system/bin/cameraserver
    setenv LD_PRELOAD /system/lib64/libcamera50mp_shim.so
```

All existing service options must remain below that declaration. If your RC
differs, merge only the `setenv` option into the existing cameraserver service.
Use [cameraserver-init.patch](cameraserver-init.patch) as the minimal change.
If the service already has `LD_PRELOAD`, combine the library paths in that
value instead of discarding the existing preload.

Keep the original `/system/bin/cameraserver` and
`/system/lib64/libcameraservice.so`. No launcher or files under `/data` are
required in the final ROM. Add the library to your image builder's ownership,
permission, and SELinux metadata using [permissions.tsv](permissions.tsv) and
[file_contexts.fragment](file_contexts.fragment).

Some unpackers extract a system-as-root image under `system/system/`. Adjust
the unpacked path so the final Android path matches the table. Rebuild and
flash the modified partition using the normal process for your ROM.

The shim itself was tested live on the device. Booting a rebuilt partition
with this init change has not yet been tested.

## Verified results

| Capture | File size | Decoding |
| --- | ---: | --- |
| Before fix, 50MP | 4,483,924 bytes | Truncated; JPEG end marker missing |
| With shim, 50MP | 11,549,478 bytes | Complete 6144 x 8192 |
| With shim, 50MP | 12,201,529 bytes | Complete 6144 x 8192 |
| After service restart, 50MP | 14,352,502 bytes | Complete 6144 x 8192 |
| Normal photo | 6,390,287 bytes | Complete 3072 x 4096 |

Image detail was present through the final rows. The buffer-too-small error
was absent for the verified successful captures. Old truncated photos are
not repaired by installing the shim.

## Check after boot

```sh
adb logcat -d -s Camera50MPShim
adb shell su -c 'cat /proc/$(pidof cameraserver)/maps | grep libcamera50mp_shim'
```

The library should be mapped from `/system/lib64/`. On a 50MP capture, expect:

```
Preserved advertised maximum-resolution JPEG 8192x6144
```

The shim retains only JPEG/BLOB dimensions explicitly advertised as an output
in the camera's maximum-resolution map and absent from its default map. Other
requests use the original rounding function.

## Removal

Remove the added `setenv LD_PRELOAD` option, or remove only this shim's path
from an existing preload list. Remove the `.so` from the image, rebuild, and
flash normally. No original camera binaries need restoring.

## Credits
and sir @ireddragonicy for reference for jepg buffering fix.
