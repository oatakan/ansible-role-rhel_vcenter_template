# oatakan.rhel_vcenter_template

Provision a **RHEL/CentOS (and compatible) VM template on VMware vCenter** from an installer ISO that already exists on a vCenter datastore.

This role focuses on the **vCenter lifecycle** (create VM → install via kickstart → optional post-config roles → convert to template → optional OVF/OVA export).
For OS “golden image” hardening and cleanup, it can optionally run the companion role [`oatakan.rhel_template_build`](https://github.com/oatakan/ansible-role-rhel_template_build) over SSH during the build.

> Note: This role uses `validate_certs: false` for VMware modules in its tasks.

## What this role does

High-level flow (see [tasks/main.yml](tasks/main.yml) and [tasks/provision.yml](tasks/provision.yml)):

1. **Preflight** ([tasks/preflight_check.yml](tasks/preflight_check.yml))
   - Validates the installer ISO exists in the datastore at `{{ datastore_iso_folder }}/{{ iso_file_name }}`.
   - Detects whether a template already exists with `template_vm_name`.
2. **Kickstart ISO (OEMDRV)**
   - Renders `ks.cfg` from [templates/](templates/) and builds a small ISO via `mkisofs` ([tasks/make_iso.yml](tasks/make_iso.yml)).
   - Uploads it to the datastore ([tasks/datastore_upload.yml](tasks/datastore_upload.yml)).
3. **Provision VM**
   - Creates/powers on the VM with two CDROMs: installer ISO + the kickstart ISO ([tasks/provision_vm.yml](tasks/provision_vm.yml)).
   - Waits for SSH to become reachable.
4. **Optional post-install configuration over SSH**
   - Adds a runtime host `template_vm` and delegates SSH roles to it ([tasks/provision.yml](tasks/provision.yml)).
   - Can run `oatakan.rhn`, `oatakan.rhel_upgrade` (updates), and `oatakan.rhel_template_build` (golden image prep).
5. **Post-config / finalize**: Shuts down guest, removes CDROM drives, optionally snapshots, converts to template, optionally exports OVF/OVA, and adds vCenter custom attributes (see [tasks/convert_to_template.yml](tasks/convert_to_template.yml)): `template_created_at` and `template_updated_at`.

## Supported systems

This repo includes kickstarts under [templates/](templates/) for `rhel7`, `rhel8`, `rhel9`, `rhel10`.
CI primarily does **lint/syntax validation** (not a real vCenter integration test).

RHEL 10 currently uses `rhel9_64Guest` by default (see `os_short_names` in [defaults/main.yaml](defaults/main.yaml)).

## Requirements

Control node requirements:

- Ansible (this role declares `min_ansible_version: 2.5` in [meta/main.yml](meta/main.yml); newer is typically required by current VMware collections)
- Ansible collections:
  - `community.vmware`
  - `community.general` (used for OVA creation in [tasks/export_ovf.yml](tasks/export_ovf.yml))
- A working VMware Python stack (commonly `pyvmomi`) available to Ansible
- `mkisofs` available on the control node (often provided by `genisoimage`)

vCenter requirements:

- Installer ISO already uploaded to the datastore (path: `{{ datastore_iso_folder }}/{{ iso_file_name }}`)
- Access to the target `datacenter`, `cluster`, and optional `resource_pool`/`folder`

## VMware credentials

The VMware tasks default to environment variables:

- `VMWARE_HOST`
- `VMWARE_USER`
- `VMWARE_PASSWORD`

(`providers.vcenter.*` in [vars/main.yml](vars/main.yml) defines location objects like datacenter/cluster/datastore, not auth by default.)

## Integration with oatakan.rhel_template_build

This role can run OS “golden image” configuration by delegating to `template_vm` over SSH.

- Enable it with `build_template: true`.
- Ensure SSH is possible using the credentials created by kickstart:
  - `local_account_username`
  - `local_account_password`
  - `template_vm_ip_address`

The delegated execution happens in [tasks/provision.yml](tasks/provision.yml) and is compatible with the design of [`oatakan.rhel_template_build`](https://github.com/oatakan/ansible-role-rhel_template_build).

## Common variables

See [defaults/main.yaml](defaults/main.yaml) for the full list. The most commonly set variables are:

| Variable | Default | Description |
| --- | ---: | --- |
| `role_action` | `provision` | `provision` or `deprovision` |
| `distro_name` | `rhel10` | Selects kickstart folder + guest ID mapping |
| `iso_file_name` | (CentOS Stream 10 ISO) | Installer ISO name in datastore |
| `datastore_iso_folder` | `iso` | Folder on datastore that contains installer ISO |
| `template_vm_name` | `centos10-stream-x64-v1` | VM/template name in vCenter |
| `template_force` | `false` | Overwrite existing template (convert to VM + delete) |
| `vcenter_datacenter` / `vcenter_cluster` | | vCenter placement |
| `vcenter_datastore` / `vcenter_folder` | | Datastore/folder placement |
| `template_vm_network_name` | `mgmt` | Port group name |
| `template_vm_ip_address` | `192.168.10.98` | Static IP used for SSH wait + delegated roles |
| `local_account_username` | `ansible` | SSH user created by kickstart |
| `local_account_password` | `""` | SSH password for delegated roles |
| `install_updates` | `true` | Runs `oatakan.rhel_upgrade` when enabled |
| `build_template` | `false` | Runs `oatakan.rhel_template_build` when enabled |
| `export_ovf` | `false` | Exports OVF/OVA after template conversion |

## Example playbook

Provision a RHEL 10 template and run `oatakan.rhel_template_build` afterwards:

```yaml
---
- name: Build a vCenter VM template via kickstart
  hosts: localhost
  gather_facts: false
  connection: local
  vars:
    distro_name: rhel10
    template_vm_name: rhel10-x64-v1

    # vCenter placement
    vcenter_datacenter: production
    vcenter_cluster: cluster1
    vcenter_resource_pool: templates
    vcenter_folder: vm-templates
    vcenter_datastore: shared-storage

    # Installer ISO already uploaded to datastore
    datastore_iso_folder: iso
    iso_file_name: rhel-10.0-x86_64-dvd.iso

    # Network (static IP is used for SSH wait/delegation)
    template_vm_network_name: mgmt
    template_vm_ip_address: 192.168.10.95
    template_vm_netmask: 255.255.255.0
    template_vm_gateway: 192.168.10.254
    template_vm_domain: example.com
    template_vm_dns_servers: [8.8.8.8, 8.8.4.4]

    # Kickstart-created SSH user for post-install roles
    local_account_username: ansible
    local_account_password: "CHANGE_ME"
    local_administrator_password: "CHANGE_ME"

    # Optional post-install hardening/build
    install_updates: true
    build_template: true

  roles:
    - role: oatakan.rhel_vcenter_template
```

Deprovision/remove an existing template:

```yaml
---
- name: Remove a vCenter template
  hosts: localhost
  gather_facts: false
  connection: local
  roles:
    - role: oatakan.rhel_vcenter_template
      vars:
        role_action: deprovision
        template_vm_name: rhel10-x64-v1
```

## Tags

The main tag groups are:

- `preflight` – datastore ISO check and template existence detection
- `provision` – kickstart ISO + VM provisioning
- `custom-roles` – SSH delegated roles (including `oatakan.rhel_template_build`)
- `post-config` – shutdown, snapshot, convert-to-template, export, cleanup

## Testing (local)

This repo’s CI runs lint and syntax checks (see [.travis.yml](.travis.yml)):

```bash
ansible-playbook tests/test.yml -i tests/inventory --syntax-check
ansible-lint .

# validate distro mapping/templates
ansible-playbook tests/test.yml -i tests/inventory --syntax-check -e "distro_name=rhel7"
ansible-playbook tests/test.yml -i tests/inventory --syntax-check -e "distro_name=rhel8"
ansible-playbook tests/test.yml -i tests/inventory --syntax-check -e "distro_name=rhel9"
ansible-playbook tests/test.yml -i tests/inventory --syntax-check -e "distro_name=rhel10"
```

## Troubleshooting

- **Installer ISO not found**: ensure the datastore path `{{ datastore_iso_folder }}/{{ iso_file_name }}` exists (see [tasks/preflight_check.yml](tasks/preflight_check.yml)).
- **Template already exists**: set `template_force: true` (or run `role_action: deprovision` first).
- **SSH delegation fails**: confirm `template_vm_ip_address` is reachable and the kickstart created `local_account_username`/`local_account_password`.
- **RHEL 10 guest ID**: defaults to `rhel9_64Guest` until VMware publishes a RHEL 10-specific ID.

## License

MIT

## Author

Orcun Atakan
