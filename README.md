# trex-generator

Snap packaging for the [Cisco TRex](https://github.com/cisco-system-traffic-generator/trex-core)
traffic generator, built from the pinned upstream source under strict
confinement.

- [Security](SECURITY.md)
- [Contributing](CONTRIBUTING.md)

## Build

The snap builds on core22. On other Ubuntu releases, build in LXD:

```sh
snapcraft pack --use-lxd
```

## Test

```sh
sudo snap install --dangerous trex-generator_*_amd64.snap
for i in network-control hardware-observe process-control system-observe; do
  sudo snap connect trex-generator:$i
done
```

Connect two kernel interfaces back to back, put a configuration under
`/var/snap/trex-generator/common`, start the server, and send traffic from
port 0 to port 1 with `trex-generator.python`. Check for AppArmor denials with
`journalctl -k | grep DENIED`.

## License

The packaging is released under the [BSD 3-Clause license](LICENSE).
TRex is released under the Apache 2.0 license; bundled third-party components
retain their own licenses.
