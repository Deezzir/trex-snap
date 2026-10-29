# trex-generator

[Cisco TRex](https://github.com/cisco-system-traffic-generator/trex-core)
traffic generator packaged as a strictly confined snap. It contains the TRex
server, the interactive console, and the Python automation client, built from
the upstream source.

The server runs in af_packet mode on kernel network interfaces. It needs no
hugepages, NIC driver binding, or host tuning.

- [Security](SECURITY.md)
- [Contributing](CONTRIBUTING.md)

## Install

```sh
sudo snap install trex-generator
sudo snap connect trex-generator:network-control
sudo snap connect trex-generator:hardware-observe
sudo snap connect trex-generator:process-control
sudo snap connect trex-generator:system-observe
```

| Interface | Used for |
|---|---|
| `network-control` | Raw packet sockets on the traffic interfaces |
| `hardware-observe` | NIC discovery through `lspci` and sysfs |
| `process-control` | Writing the server's `coredump_filter` |
| `system-observe` | Detecting another running TRex instance |

## Configure

The server reads its configuration from `/var/snap/trex-generator/common`.
Interfaces are named by kernel interface name:

```yaml
- version: 2
  interfaces: ["enp6s0", "enp7s0"]
  port_info:
    - dest_mac: 00:16:3e:67:8b:d6
      src_mac: 00:16:3e:1a:dc:00
    - dest_mac: 00:16:3e:1a:dc:00
      src_mac: 00:16:3e:67:8b:d6
  port_mtu: 1500
```

| Snap option | Default | Meaning |
|---|---|---|
| `config` | `trex_cfg.yaml` | Configuration file, relative to `/var/snap/trex-generator/common` |

```sh
sudo snap set trex-generator config=af.yaml
```

Passing `--cfg <absolute path>` overrides the option. The path must be under
`/var/snap/trex-generator/common`.

## Run

| Command | Runs |
|---|---|
| `trex-generator.t-rex-64` | TRex server (`t-rex-64`) |
| `trex-generator.console` | Interactive console |
| `trex-generator.python` | Python 3.10 with the TRex automation client importable |

```sh
sudo trex-generator.t-rex-64 -i -c 1 --no-ofed-check --no-scapy-server
sudo trex-generator.python /var/snap/trex-generator/common/script.py
```

DPDK runtime files are written under `/var/snap/trex-generator/current/run`.
Scripts and their output must live under `/var/snap/trex-generator/common`.

## Limitations

DPDK PCI mode is not supported. Strict confinement denies `/dev/vfio`,
hugetlbfs mounts, and `/proc/<pid>/pagemap`.

## License

The packaging is released under the [BSD 3-Clause license](LICENSE).
TRex is released under the Apache 2.0 license; bundled third-party components
retain their own licenses.
