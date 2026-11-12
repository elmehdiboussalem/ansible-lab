# ansible-lab

Ansible playbooks and roles that configure and harden a small Ubuntu 22.04 lab running on VMware Workstation: one controller, two web servers and one Docker host. This is a learning project, companion to the `terraform-lab` repository.

## Lab layout

| Host | Role |
|---|---|
| controller | Runs Ansible (local connection) |
| web01, web02 | nginx web servers |
| docker01 | Docker host |

Addresses are in `inventory/hosts.yml` (private lab network 192.168.100.0/24).

## Requirements

- Ansible 2.17 on the controller
- SSH key access from the controller to every host (user `mehdi`)
- `sudo` on the targets (the playbook is run with `-K`)

## Roles

| Role | What it does |
|---|---|
| `common` | Installs base packages |
| `nginx` | Installs nginx, deploys a page and a configuration, validates it |
| `docker` | Installs Docker from the official repository, manages the `docker` group |
| `security` | Creates demo accounts, hardens SSH, removes listed users from the `lxd` group |
| `firewall` | Configures UFW (SSH from the lab subnet, extra ports) |
| `updates` | Enables unattended upgrades, without automatic reboot |
| `fail2ban` | Installs and configures fail2ban |
| `docker_firewall` | Restricts access to a published Docker port through the `DOCKER-USER` chain |

## Usage

Run from the repository root, where `ansible.cfg` and the inventory are:

```bash
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml --check -K
ansible-playbook playbooks/site.yml -K
```

Use `--limit <host>` to target one machine. A second run should report `changed=0`.

## Terraform link

The `docker_firewall` role reads `generated/terraform_docker.json` (keys `host`, `name`, `port`), produced on the Terraform side. This file is not tracked by Git.

## Lessons learned

- **Docker bypasses UFW.** Published ports are not filtered by UFW; filtering belongs in the `DOCKER-USER` chain.
- **Known limitation of the Docker filter.** Rules are loaded by a systemd unit. After a container recreation, the chain was found empty and the unit missing on the test VM; the cause was not established. Always re-test the filter after recreating a container or restarting Docker.
- **Group changes are not `user` module work.** Use `gpasswd -d` guarded by a `getent` check. The `user` module without `append` would overwrite the whole group list and remove `sudo`.
- **`{{ }}` belongs to Jinja2.** In ad-hoc `ansible -a` commands, Docker `--format` templates must be run over plain SSH instead.
- **cloud-init and netplan.** To remove a DHCP address added by cloud-init, disable its network config first (`network: {config: disabled}` in `/etc/cloud/cloud.cfg.d/`), then clean `/etc/netplan`, and confirm with a reboot.
- **Unclean shutdowns can corrupt Git repositories.** Always `sync` after an important commit and power off with `sync && sudo poweroff`.

## Not tracked by Git

`generated/`, `.vault_pass`, `*.retry`, `*.log`. No secrets are stored in this repository.
