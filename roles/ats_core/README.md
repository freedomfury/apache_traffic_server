ats_core
========

Deploys an Apache Traffic Server install-prefix tarball, templates ATS config
files, installs the `trafficserver` systemd unit, and reloads config via
`traffic_ctl`.

This role consumes the artifact produced by the companion `ats_build` role
(or the prebuilt `files/ats-artifact-9.1.1.tar.gz` shipped alongside it).

Requirements
------------

- RHEL-family target (`yum` is used to install `tar`, `gzip`, and `hwloc-libs`).
- An ATS install-prefix archive — either:
  - a local file path under the role's `files/` (default:
    `files/ats-artifact-9.1.1.tar.gz`), or
  - an `http(s)://` URL.
- The archive must extract to a tree whose root matches `ats_core_home`
  (default `/opt/ats`).

Role Variables
--------------

### Deployment

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `ats_core_home`            | `/opt/ats`                        | Install prefix; the archive is extracted into its parent directory. |
| `ats_core_source_archive`  | `files/ats-artifact-9.1.1.tar.gz` | Local path or `http(s)://` URL of the artifact tarball. |
| `ats_core_source_username` | _unset_                           | Optional basic-auth user for URL fetches. |
| `ats_core_source_password` | _unset_                           | Optional basic-auth password for URL fetches. |
| `ats_core_source_checksum` | _unset_                           | Optional `get_url` checksum (e.g. `sha256:...`). |
| `ats_core_source_force`    | _unset_                           | Optional `get_url` `force:` value. |
| `ats_core_deploy_folder`   | `/var/cache/ats`                  | Where the tarball is staged before extraction. |

### Service

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `ats_core_service_state`   | `started` | `state:` passed to the `systemd` module. |
| `ats_core_service_enabled` | `true`    | `enabled:` passed to the `systemd` module. |

After templating the service unit, the role runs:

- `traffic_server -C verify_config` to validate config,
- `traffic_ctl config status`, and
- `traffic_ctl config reload` if reconfiguration is required.

### Configuration

Set `ats_core_manage_config: false` to skip all config templating. Otherwise
each of the following variables, when defined, drives a single templated
config file:

| Variable | Output file | Notes |
| -------- | ----------- | ----- |
| `ats_core_cache_config`         | `etc/cache.config`     | List of key/value rows. |
| `ats_core_hosting_config`       | `etc/hosting.config`   | List of key/value rows. |
| `ats_core_ip_allow`             | `etc/ip_allow.yaml`    | Rendered via `to_nice_yaml`. |
| `ats_core_logging`              | `etc/logging.yaml`     | Rendered via `to_nice_yaml`. |
| `ats_core_parent_config`        | `etc/parent.config`    | List of key/value rows. |
| `ats_core_plugin_config`        | `etc/plugin.config`    | List of `{name, args: [...]}`. |
| `ats_core_splitdns_config`      | `etc/splitdns.config`  | List of key/value rows. |
| `ats_core_ssl_multicert_config` | `etc/multicert.config` | List of key/value rows. |
| `ats_core_sni`                  | `etc/sni.yaml`         | Rendered via `to_nice_yaml`. |
| `ats_core_storage_config`       | `etc/storage.config`   | List of `{pathname, size, volume?, id?}`. |
| `ats_core_strategies`           | `etc/strategies.yaml`  | Rendered via `to_nice_yaml`. |
| `ats_core_volume_config`        | `etc/volume.config`    | Sourced from `ats_core_ssl_volume_config`. |
| `ats_core_remap_config`         | `etc/remap.config`     | See remap section below. |

#### Remap

`ats_core_remap_config` is a list of rule rows. Supported `type`s:

- `map`, `reverse_map`, `redirect`, `redirect_temporary` — produce
  `<type> <target> <replace>` with optional `acls:` and `plugins:`.
- `map_with_referer` — `client_url`, `origin_server_url`, `redirect_url`,
  `regexes`.
- `definefilter`, `activatefilter`, `deactivatefilter` — emit `.<type>` lines.
- `include` — emits `.include <files...>`. The listed filenames are also
  rendered into `{{ ats_core_conf_inc_dir }}` (default
  `{{ ats_core_home }}/etc/trafficserver/conf.d`) by looking up
  `ats_core_remap_includes[<basename without extension>]` and templating it
  through the same `remap.config.j2`.

`ats_core_purge_inc_dir` (default `true`) removes any file in the conf.d
directory that the role didn't just write.

Dependencies
------------

None. Pairs naturally with `ats_build` from the same collection but does not
require it.

Example Playbook
----------------

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

License
-------

MIT

Author Information
------------------

Freedom Fury &lt;freedomfury@gmail.com&gt;
