# Ansible Playbooks

A focused collection of Ansible playbooks for Linux infrastructure checks,
system inventory, credential maintenance, and Raspberry Pi patching. The
patching workflow also discovers and refreshes Docker Compose stacks.

## Repository scope

| Playbook | Purpose | Privilege | Key result |
| --- | --- | --- | --- |
| [`ping.yml`](ping.yml) | Test Ansible and SSH connectivity to inventory hosts | None | Confirms the host responds to Ansible and TCP port 22 |
| [`Linux_system_info.yml`](Linux_system_info.yml) | Gather host operating system details | None on managed hosts | Writes `linux_system_info.csv` on the control node |
| [`Root_password_rotation.yaml`](Root_password_rotation.yaml) | Set a new root password across selected hosts | `become` | Updates the root password using a SHA-512 hash |
| [`patching/patch.yml`](patching/patch.yml) | Patch Debian-based Raspberry Pi hosts and update Docker Compose stacks | `become` | Upgrades packages, pulls images, recreates stacks, verifies containers, and reboots when configured or required |

## Requirements

- Ansible installed on the control node
- An inventory containing the target hosts
- SSH access to each managed host
- Privilege escalation configured for playbooks that use `become`
- Python available on each managed host
- Debian or Ubuntu-style `apt` package management for the patching workflow
- Docker with the Compose plugin installed for `patching/patch.yml`

## Quick start

Create an inventory such as:

```ini
[linux]
server-01 ansible_host=192.0.2.10 ansible_user=admin

[raspberry_pi]
pi-01 ansible_host=192.0.2.20 ansible_user=pi
```

Check that Ansible can reach the hosts:

```bash
ansible-playbook -i inventory.ini ping.yml
```

Run a playbook against a specific inventory group with `--limit`:

```bash
ansible-playbook -i inventory.ini Linux_system_info.yml --limit linux
```

Use `--ask-become-pass` when the remote account requires a sudo password:

```bash
ansible-playbook -i inventory.ini patching/patch.yml \
	--limit raspberry_pi \
	--ask-become-pass
```

## Playbook guide

### Connectivity check

[`ping.yml`](ping.yml) runs the Ansible ping module and then checks whether SSH
port 22 is open on each target host.

```bash
ansible-playbook -i inventory.ini ping.yml
```

### Linux system inventory

[`Linux_system_info.yml`](Linux_system_info.yml) gathers facts and records each
host's name, distribution, and distribution version in
`linux_system_info.csv` on the Ansible control node.

```bash
ansible-playbook -i inventory.ini Linux_system_info.yml
```

Override the output path when needed:

```bash
ansible-playbook -i inventory.ini Linux_system_info.yml \
	-e output_file=/tmp/linux_system_info.csv
```

### Root password rotation

[`Root_password_rotation.yaml`](Root_password_rotation.yaml) prompts for and
confirms a new root password, hashes it with SHA-512, and applies it to every
selected host.

```bash
ansible-playbook -i inventory.ini Root_password_rotation.yaml \
	--limit linux \
	--ask-become-pass
```

> [!CAUTION]
> Test password rotation on a limited host group first and keep an independent
> administrative session open until the new credential has been verified.

### Raspberry Pi patching and Docker updates

[`patching/patch.yml`](patching/patch.yml) performs the following sequence:

1. Reports host and running-container information.
2. Finds `docker-compose.yml` and `docker-compose.yaml` files.
3. Runs an `apt` cache refresh, full upgrade, autoremove, and autoclean.
4. Pulls current images and recreates every discovered Compose stack.
5. Reports the resulting container state.
6. Reboots when the OS requests it, packages changed, or `reboot_always` is enabled.

Default settings live in
[`patching/group_vars/all.yml`](patching/group_vars/all.yml):

| Variable | Default | Description |
| --- | --- | --- |
| `compose_search_paths` | `[/export/docker]` | Directories searched recursively for Compose files |
| `reboot_always` | `true` | Forces a reboot even when the OS does not request one |

Run the workflow against the intended Raspberry Pi group:

```bash
ansible-playbook -i inventory.ini patching/patch.yml \
	--limit raspberry_pi \
	--ask-become-pass
```

Override settings without editing the defaults:

```bash
ansible-playbook -i inventory.ini patching/patch.yml \
	--limit raspberry_pi \
	--ask-become-pass \
	-e '{"compose_search_paths":["/opt/stacks","/srv/compose"],"reboot_always":false}'
```

> [!WARNING]
> The patching playbook upgrades system packages, recreates containers, and may
> reboot hosts. Use `--limit`, confirm the Compose search paths, and schedule an
> appropriate maintenance window before running it.

## Safer execution

Review the targeted hosts and preview changes before an operational run:

```bash
ansible-playbook -i inventory.ini playbook.yml --list-hosts
ansible-playbook -i inventory.ini playbook.yml --check --diff
```

Check mode is only a preview; shell commands and some modules may not model
their final changes completely. Replace `playbook.yml` with the playbook you
intend to run and use `--limit` whenever the inventory contains unrelated
hosts.
