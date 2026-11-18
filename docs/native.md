# Native DPDK

`trex-generator.native` starts the packaged DPDK binary directly, under strict
confinement. It does not mount hugepages, load modules or bind NICs. Use a
dedicated NIC or VF already bound to `vfio-pci`, with a working IOMMU; do not
unbind a management or shared OVS uplink. Unsafe no-IOMMU mode is not supported.
Mellanox `mlx4`/`mlx5` drivers are not included.

Native traffic is validated with the `net_virtio` PMD using VFIO and an enabled
IOMMU. Other NIC/PMD combinations need hardware-specific validation.

## Host preparation

Prepare the device and reserve hugepages outside the snap. The kernel must
support VFIO and IOMMU. Reserve sufficient hugepages at boot if runtime
allocation fails. Mount a single hugetlbfs at `/dev/hugepages`; do not stack
mounts with different page sizes at that path.

Confirm the selected PCI address, its `vfio-pci` driver and its IOMMU group:

```sh
pci=0000:05:00.0  # replace with the dedicated device
readlink /sys/bus/pci/devices/$pci/driver
group=$(basename "$(readlink /sys/bus/pci/devices/$pci/iommu_group)")
ls -l /dev/vfio/$group
cat /sys/module/vfio/parameters/enable_unsafe_noiommu_mode  # must be N
findmnt /dev/hugepages
```

All devices in the selected IOMMU group must be suitable for exclusive VFIO use.
VLAN traffic also requires the connected switch or virtual bridge to allow the
selected tags.

## Device permission

Connect the `vfio` plug to a `custom-device` slot with the identifier
`custom-device: trex-vfio`, supplied by the machine's gadget or a locally
prepared access snap. The slot grants only the selected group. For a local
installation, create this declarative metadata after setting and verifying
`group` above:

```sh
mkdir -p vfio-access/meta
cat > vfio-access/meta/snap.yaml <<EOF
name: trex-vfio-access
version: '1'
summary: Access to the dedicated TRex VFIO group
description: Local device permission for an operator-selected IOMMU group.
confinement: strict
grade: devel
slots:
  vfio:
    interface: custom-device
    custom-device: trex-vfio
    devices:
      - /dev/vfio/vfio
      - /dev/vfio/$group
    files:
      read:
        - /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
        - /sys/kernel/iommu_groups/$group
        - /sys/kernel/iommu_groups/$group/devices
EOF
snap pack vfio-access
sudo snap install --dangerous trex-vfio-access_1_all.snap
sudo snap connect trex-generator:vfio trex-vfio-access:vfio
sudo snap connect trex-generator:hugepages-control
```

Also connect the four standard interfaces listed in the README. Recreate the
access snap if the device's group number changes. Store distribution of a
custom-device slot requires the corresponding Snap Store declaration; a local
`--dangerous` install does not establish Store approval.

## Configuration and launch

Save the following as `/var/snap/trex-generator/common/native.yaml`, replacing
the PCI address and CPU IDs. The example uses one real port and one dummy port,
512 MiB of DPDK memory and an explicit 1500-byte MTU. Select an MTU the NIC
supports and reserve enough hugepages for the selected memory limit.
`--huge-unlink=always` lets pages return to the pool when the process exits.

```yaml
- version: 2
  interfaces: ["0000:05:00.0", "dummy"]
  port_mtu: 1500
  prefix: trex-native
  low_end: true
  limit_memory: 512
  ext_dpdk_opt: ["--huge-dir", "/dev/hugepages", "--huge-unlink=always"]
  platform:
    master_thread_id: 0
    latency_thread_id: 1
    dual_if:
      - socket: 0
        threads: [2]
```

```sh
sudo trex-generator.native -i -c 1 --no-scapy-server --no-ofed-check \
  --cfg /var/snap/trex-generator/common/native.yaml
```

Both `--cfg path` and `--cfg=path` select an explicit configuration. Without
either, the command uses the snap's `config` setting, falling back to
`/var/snap/trex-generator/common/trex_cfg.yaml`.

Use the same console and Python client as the AF_PACKET server. Each PCI
device is owned by one TRex process. Separate processes require separate
devices and distinct CPU, RPC and DPDK-prefix assignments.

## Startup failures

- `Cannot open VFIO container` or a VFIO `Permission denied`: check the
  `trex-generator:vfio` connection and that its slot names the current group.
- `Could not find requested interface`: check the PCI address, `vfio-pci`
  binding, device permissions and whether the packaged PMD supports the NIC.
- Hugepage allocation failure: check the reserved page size, free pages, single
  hugetlbfs mount and `trex-generator:hugepages-control` connection.
- Unsupported MTU: set `port_mtu` to a value supported by the selected NIC.
