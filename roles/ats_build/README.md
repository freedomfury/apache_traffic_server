ats_build
=========

Ansible role that automates the build process of Apache Traffic Server from source code.

Requirements
------------

Url of on the source file to build and install binaries.

Role Variables
--------------

| Variable                   | Type        | Value          |
| -----------                | ----------- |------------    |
| ats_build_source_url       | default     | undef          |
| ats_build_source_options   | default     | {}             |
| ats_build_force            | default     | false          |
| ats_build_directory        | default     | /opt/ats_build |
| ats_build_configure_prefix | default     | /opt/ats       |
| ats_build_artifact_prefix  | default     | ats-artifact   |



Dependencies
------------
Role has no dependencies.

Example Playbook
----------------
At least minimal facts are required for role to make the distinction between operating systems.
```
---
- name: Build Apache Traffic Server from source
  hosts: all
  gather_subset: min
  tasks:
    - name: "Include ats_build"
      include_role:
        name: "ats_build"
      vars:
        ats_build_source_path: https://github.com/apache/trafficserver/archive/refs/tags/9.1.1.tar.gz
        ats_build_force: true
```

License
-------

MIT

Author Information
------------------
Freedom Fury
