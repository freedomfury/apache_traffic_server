# frefur.apache_traffic_server

Ansible collection for building, installing, and configuring [Apache Traffic
Server](https://trafficserver.apache.org/). Targets CentOS 7 / 8 (and AlmaLinux
8 via the molecule scenarios).

The collection is split into two roles so the build step can run on a
throwaway host and the resulting artifact can be deployed elsewhere:

| Role | Purpose |
| ---- | ------- |
| `frefur.apache_traffic_server.ats_build` | Pulls an ATS source tarball, installs build deps, runs `autoreconf` / `configure` / `make` / `make install` into a prefix, and produces a redistributable tarball of that prefix. |
| `frefur.apache_traffic_server.ats_core`  | Drops the build artifact onto a target host, renders ATS config files from variables, installs a `trafficserver.service` unit, and reloads config via `traffic_ctl`. |

A pre-built `ats-artifact-9.1.1.tar.gz` is bundled under
`roles/ats_core/files/` so `ats_core` can be exercised without first running
`ats_build`.

## Requirements

- Ansible 2.9+ on the controller.
- Target hosts: CentOS 7, CentOS 8, or AlmaLinux 8 (RHEL-family with `yum`).
- `ats_build` enables the `powertools` repo on EL8 and uses `devtoolset-7` (SCL)
  on EL7.
- For local testing, the molecule scenarios use the **lxd** driver, not docker
  — despite `molecule-docker` being listed in `requirements.txt`.

Python tooling for development (linting / molecule) is pinned in
`requirements.txt`. Versions there are old (Ansible 2.9, molecule 3.5); treat
them as a reference rather than a recommendation.

## Installation

```sh
ansible-galaxy collection install git+https://github.com/<user>/apache_traffic_server.git
```

Or build and install locally from a checkout:

```sh
ansible-galaxy collection build
ansible-galaxy collection install frefur-apache_traffic_server-*.tar.gz
```

## Roles

### `ats_build`

Builds ATS from a source tarball URL and produces an install-prefix archive at
`{{ ats_build_directory }}/{{ ats_build_artifact_prefix }}-<basename>.tar.gz`.

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `ats_build_source_url`       | `null`           | URL to the ATS source tarball. **Required.** |
| `ats_build_source_options`   | `{}`             | Extra args merged into the `get_url` task (e.g. `checksum`, auth). |
| `ats_build_force`            | `false`          | When `true`, removes `ats_build_directory` first to force a clean build. |
| `ats_build_directory`        | `/opt/ats_build` | Workspace where the source is downloaded and built. |
| `ats_build_configure_prefix` | `/opt/ats`       | `--prefix=` passed to `./configure` (also what gets archived). |
| `ats_build_artifact_prefix`  | `ats-artifact`   | Filename prefix for the produced tarball. |

Per-OS package lists and SCL wrappers live in
`roles/ats_build/vars/os/{centos7,centos8}.yml`.

Example:

```yaml
- name: Build Apache Traffic Server from source
  hosts: builders
  gather_subset: min
  tasks:
    - name: Run ats_build
      ansible.builtin.include_role:
        name: frefur.apache_traffic_server.ats_build
      vars:
        ats_build_source_url: https://github.com/apache/trafficserver/archive/refs/tags/9.1.1.tar.gz
        ats_build_force: true
```

### `ats_core`

Deploys an ATS install-prefix tarball, templates config files, installs and
manages the `trafficserver` systemd service, and validates / reloads config.

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `ats_core_home`             | `/opt/ats`                            | Install prefix (matches `ats_build_configure_prefix`). |
| `ats_core_source_archive`   | `files/ats-artifact-9.1.1.tar.gz`     | Local path or `http(s)://` URL of the artifact tarball. |
| `ats_core_source_username`  | _unset_                               | Optional basic-auth user when fetching from a URL. |
| `ats_core_source_password`  | _unset_                               | Optional basic-auth password. |
| `ats_core_source_checksum`  | _unset_                               | Optional `get_url` checksum string (e.g. `sha256:...`). |
| `ats_core_source_force`     | _unset_                               | Optional `get_url` `force:` value. |
| `ats_core_deploy_folder`    | `/var/cache/ats`                      | Where the tarball lands before extraction. |
| `ats_core_manage_config`    | `true`                                | Run the config templating step. |
| `ats_core_conf_inc_dir`     | `{{ ats_core_home }}/etc/trafficserver/conf.d` | Output dir for `remap.config` includes. |
| `ats_core_purge_inc_dir`    | `true`                                | Delete unmanaged files from `conf.d`. |
| `ats_core_service_state`    | `started`                             | `state:` passed to `systemd`. |
| `ats_core_service_enabled`  | `true`                                | `enabled:` passed to `systemd`. |

Each ATS config file is templated only when its corresponding variable is
defined. Provide whichever you need:

| Variable | Renders to | Template |
| -------- | ---------- | -------- |
| `ats_core_cache_config`         | `etc/cache.config`        | `generic.config.j2` |
| `ats_core_hosting_config`       | `etc/hosting.config`      | `generic.config.j2` |
| `ats_core_ip_allow`             | `etc/ip_allow.yaml`       | `generic.yaml.j2`   |
| `ats_core_logging`              | `etc/logging.yaml`        | `generic.yaml.j2`   |
| `ats_core_parent_config`        | `etc/parent.config`       | `generic.config.j2` |
| `ats_core_plugin_config`        | `etc/plugin.config`       | `plugin.config.j2`  |
| `ats_core_splitdns_config`      | `etc/splitdns.config`     | `generic.config.j2` |
| `ats_core_ssl_multicert_config` | `etc/multicert.config`    | `generic.config.j2` |
| `ats_core_sni`                  | `etc/sni.yaml`            | `generic.yaml.j2`   |
| `ats_core_storage_config`       | `etc/storage.config`      | `storage.config.j2` |
| `ats_core_strategies`           | `etc/strategies.yaml`     | `generic.yaml.j2`   |
| `ats_core_volume_config`        | `etc/volume.config`       | `generic.config.j2` |
| `ats_core_remap_config`         | `etc/remap.config` (+ optional `conf.d/*` includes via `ats_core_remap_includes`) | `remap.config.j2` |

`ats_core_records_config` is consumed by `records.config.j2` (scope/name/type/value
rows) but is not currently wired up in `tasks/configure.yml`.

Example:

```yaml
- name: Deploy Apache Traffic Server
  hosts: ats
  tasks:
    - name: Run ats_core
      ansible.builtin.include_role:
        name: frefur.apache_traffic_server.ats_core
      vars:
        ats_core_storage_config:
          - pathname: /var/cache/trafficserver
            size: 256M
        ats_core_remap_config:
          - type: map
            target: http://example.test/
            replace: http://origin.example.test/
```

## Testing

Each role ships a molecule scenario under `roles/<role>/molecule/default/`
using the **lxd** driver. From a role directory:

```sh
cd roles/ats_build
molecule test
```

`ats_build/molecule/default/verify.yml` runs `traffic_server --version`,
starts the service, waits for `:8080`, and stops it again.
`ats_core/molecule/default/verify.yml` is a placeholder that just asserts
`true`.

## Layout

```
.
├── galaxy.yml
├── plugins/                 # empty placeholder
├── requirements.txt         # python tooling for dev
├── requirements.yml         # empty
├── scripts/tasks.py         # empty
└── roles/
    ├── ats_build/
    └── ats_core/
        └── files/ats-artifact-9.1.1.tar.gz   # vendored prebuilt artifact
```

## License

MIT (per role metadata). `galaxy.yml` declares `GPL-2.0-or-later` for the
collection as a whole — the discrepancy is unresolved.

## Author

Freedom Fury &lt;freedomfury@gmail.com&gt;
