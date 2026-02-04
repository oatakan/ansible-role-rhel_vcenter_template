# Copilot instructions (AI coding agent)

## Repo purpose (big picture)
This repository is an **Ansible role** (`oatakan.rhel_vcenter_template`) that **builds a RHEL/CentOS VM template on VMware vCenter** from an installer ISO that already exists on a vCenter datastore. It generates a small **kickstart “OEMDRV” ISO**, uploads it to the datastore, attaches it as an extra CDROM, provisions a VM, optionally runs post-install roles over SSH, then converts the VM to a template (and optionally exports OVF/OVA).

## How the role runs (read these files first)
- Entry point: [tasks/main.yml](../tasks/main.yml)
- Always runs: [tasks/preflight_check.yml](../tasks/preflight_check.yml)
  - Checks the installer ISO exists on the datastore (`datastore_iso_folder` + `iso_file_name`).
  - Detects whether `template_vm_name` already exists as a template (`template_found`).
- Provision flow: [tasks/provision.yml](../tasks/provision.yml)
  - Kickstart ISO creation + upload: [tasks/make_iso.yml](../tasks/make_iso.yml) → [tasks/datastore_upload.yml](../tasks/datastore_upload.yml)
  - VM creation (async + wait): [tasks/provision_vm.yml](../tasks/provision_vm.yml)
  - Adds a runtime host named `template_vm` and runs delegated SSH tasks/roles against it.
  - Post-config: shutdown + remove CDROM, optional snapshot, convert to template, optional OVF export.
- Deprovision/remove flow:
  - `remove_template` is derived in [vars/main.yml](../vars/main.yml) (true when `role_action=deprovision` or `template_force=true`).
  - Removal happens via [tasks/convert_to_vm.yml](../tasks/convert_to_vm.yml) + [tasks/remove_vm.yml](../tasks/remove_vm.yml) when `template_found`.

## Key variables and where they come from
- Defaults are in [defaults/main.yaml](../defaults/main.yaml).
- Derived variables and the canonical `template` structure are in [vars/main.yml](../vars/main.yml).
  - `distro_name` → `linux_ks_folder` + `template_vm_guest_id` via `os_short_names` in defaults.
  - `iso_file` points to the uploaded kickstart ISO on the datastore.

## VMware credentials (important)
Most VMware modules use:
- `VMWARE_HOST`, `VMWARE_USER`, `VMWARE_PASSWORD` (env vars)

The `providers.vcenter.*` mapping in [vars/main.yml](../vars/main.yml) does **not** define hostname/user/password by default, so real runs typically rely on those env vars.

## Templates (kickstart)
Kickstart templates live in:
- [templates/](../templates/) (e.g. `templates/rhel10/ks.cfg.j2`)

When editing kickstarts, note they reference role vars like `local_account_username`, `local_account_password`, `local_administrator_password`, `template.networks[0].ip`, and `template_selinux_enabled`.

## Tags and execution patterns
- Tags are applied at the include sites in [tasks/main.yml](../tasks/main.yml) and [tasks/provision.yml](../tasks/provision.yml): `preflight`, `provision`, `custom-roles`, `post-config`.
- Long-running vCenter operations use the pattern `async: 7200` + `poll: 0` and then `ansible.builtin.async_status` with retries (see [tasks/provision_vm.yml](../tasks/provision_vm.yml), [tasks/stop_vm.yml](../tasks/stop_vm.yml), [tasks/remove_vm.yml](../tasks/remove_vm.yml), [tasks/export_ovf.yml](../tasks/export_ovf.yml)). Keep this pattern when adding similar operations.

## Local dev / CI commands (from Travis)
- Syntax check: `ansible-playbook tests/test.yml -i tests/inventory --syntax-check`
- Lint: `ansible-lint .` (config in [.ansible-lint](../.ansible-lint))
- Travis creates an `ansible.cfg` with `roles_path=../` before running syntax checks (see [.travis.yml](../.travis.yml)).
