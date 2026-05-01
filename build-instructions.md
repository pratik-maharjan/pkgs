## Phase 1: Build the Kernel with USB/IP Modules (pkgs repo)

### Step 1.1 — Clone pkgs and checkout the exact ref for v1.11.5

```bash
git clone https://github.com/siderolabs/pkgs.git
cd pkgs

# Talos v1.11.5 pins pkgs at v1.11.0-29-gaee690b
git checkout aee690b
```

> Talos v1.11.5's release notes show
> `siderolabs/pkgs v1.11.0-29-gaee690b`. Using the exact ref ensures that the
> custom kernel is binary-compatible with the rest of the Talos v1.11.5 stack.

### Step 1.2 — Enable USB/IP in the kernel config

```bash
vim kernel/build/config-amd64
```

Search for `USBIP`. Add/modify these lines:

```
CONFIG_USBIP_CORE=m
CONFIG_USBIP_VHCI_HCD=m
CONFIG_USBIP_VHCI_HC_PORTS=8
CONFIG_USBIP_VHCI_NR_HCS=1
```

Verify prerequisites are already enabled:

```bash
grep "CONFIG_USB=" kernel/build/config-amd64      # Should be: =y
grep "CONFIG_NET=" kernel/build/config-amd64       # Should be: =y
```

### Step 1.3 — Create the usbip package in pkgs

```bash
mkdir -p usbip
```

Create `usbip/pkg.yaml`:

```yaml
name: usbip
variant: scratch
shell: /bin/bash
dependencies:
  - stage: base
  - stage: kernel
steps:
  - install:
      - |
        KVER=$(ls /usr/lib/modules/)
        mkdir -p /rootfs/usr/lib/modules/${KVER}/kernel/drivers/usb/usbip/
        cp /usr/lib/modules/${KVER}/kernel/drivers/usb/usbip/usbip-core.ko \
           /rootfs/usr/lib/modules/${KVER}/kernel/drivers/usb/usbip/
        cp /usr/lib/modules/${KVER}/kernel/drivers/usb/usbip/vhci-hcd.ko \
           /rootfs/usr/lib/modules/${KVER}/kernel/drivers/usb/usbip/
        depmod -b /rootfs/usr ${KVER}
finalize:
  - from: /rootfs
    to: /
```

> **Critical Note**: The `- stage: kernel` dependency is required. Without it, the
> build container has no access to `/usr/lib/modules/` and you'll get
> `ls: cannot access '/usr/lib/modules/': No such file or directory`.
>
> Also note:
> - Use `/usr/lib/modules/` (not `/lib/modules/`) — this is the Talos path
> - Use `shell: /bin/bash` (not `/bin/sh`) — matches the pattern of other extensions
> - `depmod -b /rootfs/usr` (not `/rootfs`) — so depmod looks at `/rootfs/usr/lib/modules/`

### Step 1.4 — Register in .kres.yaml

Edit `.kres.yaml` in the repo root. Find the targets list under the kernel-dependent
packages section and add `usbip`:

```yaml
spec:
  targets:
    # ... existing targets ...
    # - kernel & dependent packages (out of tree kernel modules)
    - usbip
```

Regenerate the Makefile:

```bash
make rekres
```

### Step 1.5 — Build kernel + usbip package

Prune first to ensure enough space:

```bash
kubectl exec -n buildkitd <buildkitd-pod> -- buildctl prune --all
```

Then build:

```bash
make kernel usbip \
  REGISTRY=${REGISTRY} \
  PLATFORM=${PLATFORM} \
  PUSH=${PUSH}
```

> **Build time**: 30–90 minutes. If you hit "No space left on device" during the
> kernel module install step (around `strip:` or `cp: error copying ... i915.ko`),
> your buildkitd PVC is too small.

When done, capture the image refs from output:

```bash
# Example output:
# => pushing manifest for company.jfrog.io/siderolabs/kernel:v1.11.0-29-gaee690b-dirty...
# => pushing manifest for company.jfrog.io/siderolabs/usbip:v1.11.0-29-gaee690b-dirty...

export KERNEL_IMAGE="company.jfrog.io/siderolabs/kernel:v1.11.0-29-gaee690b-dirty"
export PKG_IMAGE="company.jfrog.io/siderolabs/usbip:v1.11.0-29-gaee690b-dirty"
```

> **Note**: The `-dirty` suffix appears because you modified the kernel config.
> This is normal and expected.
