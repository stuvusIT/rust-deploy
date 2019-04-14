# rust-deploy

This role installs [Rust](https://www.rust-lang.org/) crates (optionally building them using 
[Cargo](https://doc.rust-lang.org/cargo/)) and creates [systemd](https://wiki.debian.org/systemd)
services running the installed binaries.

## Requirements

An apt based system like [Ubuntu](https://www.ubuntu.com/) or [Debian](https://www.debian.org/).

## Updating

This role can not only install rust crates, but also update their version.
For more information, see [crate objects](#crate-objects).

## Role Variables

| Name                   |  Default   | Description                                               |
| :--------------------- | :--------: | :-------------------------------------------------------- |
| `rust_deploy_crates`   |    `{}`    | Dictionary with [crate object](#crate-objects) values     |
| `rust_deploy_services` |    `{}`    | Dictionary with [service object](#service-objects) values |
| `global_cache_dir`     | _required_ | Download target directory when using `archive_url`.       |

The distinction between crates and services exists because multiple services can run the same
binary.
That binary then only needs to be installed once (i.e. by one crate). 

### Crate Objects

A crate object is a dictionary which can contain the following keys.

| Key                  |  Default   | Description                                         |
| :------------------- | :--------: | :-------------------------------------------------- |
| `archive_url`        | _optional_ | See [source specification](#source-specification)   |
| `binary_url`         | _optional_ | See [source specification](#source-specification)   |
| `build_dependencies` |    `[]`    | `apt` packages installed before building the crate. |
| `checksum`           | _required_ | Checksum of the downloaded archive or binary.       |
| `github_repo`        | _optional_ | See [source specification](#source-specification)   |
| `version`            | _required_ | Determines the installation target directory.       |

The binary is installed to
`/opt/{{ name }}-{{ version[:10] }}-{{ checksum.split(':')[-1][:40] }}/bin/{{ name }}`
and a symlink targeting it is created in `/usr/local/bin`.
This way, changing the version and re-deploying the role leaves the old version in place but links
the symlink to the new version.

For `checksum` it is recommended to use [SHA256](https://en.wikipedia.org/wiki/SHA-2).
Example:

```yml
checksum: sha256:ef7f0f5eaa31cf8a898c9d725b9f694f6c72556f667bc9f30fcd5eb2e3d3b8a1
```

One way to obtain that checksum is to insert a fake checksum (for example the one above), run
Ansible and take the actual checksum from the Ansible error.
The reason why `checksum` is _required_ is to enforce purity, i.e. the same rust-deploy
configuration always yields the same installed software.

### Source Specification

There are multiple ways to install a crate.
Chosing among them is done by providing at exactly one of the following keys per
[crate object](#crate-objects).
Actually, multiple of those keys can be provided, but then only the one listed here first is
considered.

* `binary_url`:
  Download a precompiled binary from that URL.
  When using this, the build dependencies aren't installed.
* `archive_url`:
  Build from source.
  The source is downloaded as an archive from that URL.
  The downloaded archive must contain a folder named `{{ crate_name }}-{{ version }}` which contains
  in turn the source that can be compiled with `cargo build`.
  The compilation must yield a binary also named `{{ crate_name }}` which is then installed.
* `github_repo`: 
  Short form for `archive_url: https://github.com/{{ github_repo }}/archive/{{ version }}.tar.gz`.

Hereby `crate_name` is the key of the crate object itself in `rust_deploy_crates` and `version` is
the respective value inside that crate object.

### Service Objects

A service object is a dictionary which can contain the following keys.
All of them are _required_.

| Key      |            | Description                                               |
| :------- | :--------: | :-------------------------------------------------------- |
| `binary` | _required_ | The service runs the binary `/usr/local/bin/{{ binary }}` |
| `config` | _required_ | See [service configuration](#service-configuration)       |

### Service Configuration

Each service deployed by this role has an own working directory `/etc/{{ service_name }}`.
The value of the `config` key in a [service object](#service-objects) is converted to TOML and then
written to the file `/etc/{{ service_name }}/{{ binary }}.toml`.

Hereby `service_name` is the key of the service object itself in `rust_deploy_services` and `binary`
is the respective value inside that service object.

## Example Playbook

The following playbook works if `~/.cache/stuvus` exists on the localhost.

```yml
- hosts: rust-deploy
  roles:
    - role: rust-deploy
      global_cache_dir: "{{ lookup('env', 'HOME') }}/.cache/stuvus"

      rust_deploy_crates:
        baz-out:
          github_repo: haslersn/baz-out
          version: 94658024e12a1d9e418c31e6ce6b7ed0b93263d1
          checksum: sha256:ef7f0f5eaa31cf8a898c9d725b9f694f6c72556f667bc9f30fcd5eb2e3d3b8a0
        castle:
          github_repo: haslersn/castle
          version: 6e2c3ebce122ea1a6ad877ca0e9f4a9a4ab6adb6
          checksum: sha256:fc3f6d0faa808bd353930b7bed899c4bbeec9539c1496c1b69449ed9075071a3
        noorton:
          github_repo: haslersn/noorton
          version: 70e9d18953c7d5a47a3fb3d00f608037ae997936
          checksum: sha256:86bb0cf2c27929d8a818b7052295535edc64a4ec7df8c7bbcc183bb4d354c3ca

      rust_deploy_services:
        baz-out01:
          binary: baz-out
          config:
            client:
              endpoint: http://localhost:8020/castle/lock
            policy:
              lock_after_seconds: 900
        castle01:
          binary: castle
          config:
            expander_device: /dev/spidev0.0
            output_pins:
              green_leds: [ 1, 4 ]
              red_leds: [ 2, 3 ]
              lock: 0
            input_pins:
              hinge: 2
            server:
              mount_point: /castle
              port: 8020
        noorton01:
          binary: noorton
          config:
            expander_device: /dev/spidev0.0
            client:
              endpoint: http://localhost:8020/castle/lock?toggle
            input_pins:
              switches: [ 1 ]
```
