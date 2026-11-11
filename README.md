# ansible-ubuntu-baseline

An Ansible role that brings Ubuntu 22.04 and 24.04 servers on AWS up to a security baseline and keeps them there. It connects **through AWS Systems Manager**, so there's no SSH port, no bastion and no keys to distribute.

| Area | What the role enforces |
|---|---|
| **SSH** | Key-only login, no root, no password or keyboard-interactive auth, no agent/X11/TCP forwarding, `MaxAuthTries 3`, verbose logging, a login banner, and modern KEX/ciphers/MACs only (including the post-quantum `sntrup761x25519`). Written as an `sshd_config.d` drop-in that loads *before* cloud-init's, and validated with `sshd -t` before any restart. |
| **Patching** | `unattended-upgrades` for security pockets only (including ESM), kernel clean-up, and an optional timed reboot. |
| **Kernel** | No ICMP redirects or source routing, reverse-path filtering, SYN cookies, `kptr_restrict`, `dmesg_restrict`, Yama ptrace scope, protected links, and no SUID core dumps. |
| **Audit** | `auditd` rules for identity files, sudoers and sshd config, root commands run by real users (`auid>=1000`), and writes or renames under **application directories**. When an app directory "moves", the logs say who moved it and when. |
| **Time** | `chrony` pointed at the **Amazon Time Sync Service** (169.254.169.123). |
| **Agents** | Optional: **SSM Agent** (snap) and **Wazuh agent**, registered to your manager and agent group at install time. |
| **Hygiene** | Removes clear-text clients (telnet, rsh, ftp) and installs `needrestart`. |

## Run it over SSM

```yaml
# inventories/example/group_vars/all.yml
ansible_connection: community.aws.aws_ssm
ansible_aws_ssm_region: eu-west-1
ansible_aws_ssm_bucket_name: my-ansible-ssm-transfer
baseline_install_wazuh_agent: true
baseline_wazuh_manager: wazuh-manager.internal.example
baseline_ssh_allow_groups: [ops]
```

```bash
pip install ansible boto3
ansible-playbook site.yml --check --diff     # see what would change
ansible-playbook site.yml                    # apply
```

The control node needs the Session Manager plugin and IAM permission for `ssm:StartSession` on the targets. The bucket is used to transfer module files.

## Key variables

| Variable | Default | |
|---|---|---|
| `baseline_ssh_allow_groups` | `[]` | Restrict SSH to these groups |
| `baseline_ssh_allow_tcp_forwarding` | `false` | Turn on only for hosts that need tunnels |
| `baseline_unattended_security_only` | `true` | Limit automatic upgrades to security updates |
| `baseline_unattended_auto_reboot` | `false` | Reboot automatically after updates, at `baseline_unattended_reboot_time` |
| `baseline_audit_watch_dirs` | `[/var/www, /opt]` | Only directories that exist get a watch |
| `baseline_ntp_servers` | `[169.254.169.123]` | |
| `baseline_install_ssm_agent` / `baseline_install_wazuh_agent` | `false` | |

## Tests

[Molecule](https://ansible.readthedocs.io/projects/molecule/) runs the role in systemd containers for **Ubuntu 24.04 and 22.04**. CI runs this on every push. Each test:

1. **Converges** the role.
2. Runs it a **second time and requires zero changes**, which proves idempotence.
3. **Verifies** the result: the effective `sshd -T` settings (no root, no passwords, no forwarding, no CBC ciphers), the audit rules (including watches only on directories that exist), the security-only update policy, the kernel settings, and that clear-text clients are gone.

Kernel, audit and clock settings are written in containers but only *loaded* on real hosts, because containers can't change them.

```bash
pip install ansible ansible-lint molecule "molecule-plugins[docker]"
ansible-lint            # production profile
molecule test
```

## License

MIT
