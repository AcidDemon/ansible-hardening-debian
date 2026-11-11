# ansible-hardening-debian

![Ansible](https://img.shields.io/badge/Ansible-2.16%2B-EE0000?logo=ansible&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-13%20trixie-A81D33?logo=debian&logoColor=white)
![Hardening](https://img.shields.io/badge/security-hardening-4B275F)
![2FA](https://img.shields.io/badge/2FA-TOTP%20%2B%20FIDO2-005571)
![Tested with Vagrant](https://img.shields.io/badge/tested%20with-Vagrant-1868F2?logo=vagrant&logoColor=white)

[![License: MIT](https://img.shields.io/github/license/AcidDemon/ansible-hardening-debian?color=blue)](LICENSE)
[![Last commit](https://img.shields.io/github/last-commit/AcidDemon/ansible-hardening-debian)](https://github.com/AcidDemon/ansible-hardening-debian/commits/main)
[![Repo size](https://img.shields.io/github/repo-size/AcidDemon/ansible-hardening-debian)](https://github.com/AcidDemon/ansible-hardening-debian)
[![GitLab mirror](https://img.shields.io/badge/mirror-GitLab-FC6D26?logo=gitlab&logoColor=white)](https://git.inviziblenet.work/AcidDemon/ansible-hardening-debian)

Ansible playbooks and roles to harden a freshly installed Debian 13 host — covering
user accounts, SSH, firewall, intrusion detection, auditing, and two-factor
authentication in a single idempotent run.

---

## What it deploys

| Role | What it does |
|---|---|
| `os_base` | Sets locale, timezone, hostname, and installs base packages |
| `time_sync` | Configures chrony for NTP synchronisation |
| `accounts` | Enforces password-quality policy (pwquality) and optional faillock account-locking |
| `ssh_hardening` | Drops `10-hardening.conf` into `sshd_config.d`: disables password auth, pins the `sshusers` group, restricts ciphers/MACs/KEX (optional) |
| `crowdsec` | Installs CrowdSec, enables `sshd` + `linux` collections, whitelists `management_cidrs` |
| `fail2ban` | Installs fail2ban with SSH jail; ignores `management_cidrs` (via `fail2ban_ignoreip`) |
| `auto_updates` | Enables unattended-upgrades for security updates with optional auto-reboot |
| `sysctl_hardening` | Applies kernel hardening via sysctl (IP forwarding off, ICMP redirects off, etc.) |
| `auditd` | Deploys auditd with CIS-aligned rules; optionally immutable rule set |
| `aide` | Installs AIDE and initialises the file-integrity database |
| `lynis` | Installs Lynis for on-demand security auditing |
| `hardening_misc` | Disables coredumps, enables persistent journald, restricts cron, configures msmtp for mail |
| `ssh_2fa` | Adds TOTP (google-authenticator-libpam) and FIDO2 (sk-ed25519) 2FA tiers via PAM; the `ansible` deploy account is exempt by design |

---

## Two-phase model

### Phase 1 — `playbooks/bootstrap.yml`

Ensures `python3` is present. **Account/group/key creation (admin + deploy users,
`sshusers`/`fido2users` groups, sudoers) and the host firewall are owned by
`ansible-base-debian`**, which runs before this repo in the `base → hardening → app`
order. This repo no longer manages users or nftables.

### Phase 2 — `site.yml` (→ `playbooks/harden.yml`)

Runs as the `ansible` deploy account (key-only, no password prompt).
Applies the hardening roles in lockout-safe order (the firewall is base-debian's):

1. `os_base`, `time_sync`, `accounts` — baseline OS and account policy
2. `ssh_hardening` → **connectivity check** (verifies SSH still reachable)
3. `crowdsec`, `fail2ban`, `auto_updates`, `sysctl_hardening`, `auditd`, `aide`, `lynis`, `hardening_misc`
4. `ssh_2fa` → **connectivity check** (last: highest lockout risk)

**Lockout safety mechanisms:**

- The dead-man's-switch (`harden_safe_rollback: true`) arms a `systemd-run`
  transient timer at the start of the run. If Ansible loses connectivity and
  cannot disarm it, the timer fires after `harden_rollback_minutes` minutes
  (default: 20) and auto-reverts the SSH drop-ins to their pre-run state. (The
  firewall is base-debian's; the rollback never touches nftables.)
- Each high-risk role (ssh_hardening, ssh_2fa) is followed by a
  connectivity check task that fails fast if the control host can no longer reach
  the managed node over SSH.
- Each SSH config is syntax-checked (`sshd -t`) via the template `validate:`
  clause before the file is written and the service reloaded — a bad config is
  rejected before it can take effect.

---

## Setup

### 1. Install required collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Place SSH keys

| File | Purpose |
|---|---|
| `files/ssh/acid.pub` | Admin's SSH public key — authorised for the `acid` account |
| `files/ssh/deploy` | Deploy private key — used by Ansible to connect as `ansible` |
| `files/ssh/deploy.pub` | Deploy public key — installed for the `ansible` account |

Generate the deploy keypair if not present:

```bash
ssh-keygen -t ed25519 -f files/ssh/deploy -N "" -C "ansible-deploy"
```

### 3. Set `management_cidrs`

Edit `inventory/group_vars/all/main.yml` and set `management_cidrs` to the
CIDR(s) of your control host's source network:

```yaml
management_cidrs:
  - 10.0.0.0/8
```

This single variable is consumed by both of this repo's enforcement points:
CrowdSec whitelists and fail2ban `ignoreip`. (base-debian consumes the same
variable for its nftables SSH allow rule.) If your control host's IP is not
covered here, phase 2 will lock you out.

### 4. Encrypt the vault

```bash
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

---

## Running

### Local Vagrant

```bash
# Bring up the Debian 13 VM
vagrant up

# Phase 1 — bootstrap (creates acid + ansible accounts, installs keys)
ansible-playbook -i inventory/vagrant-bootstrap.yml playbooks/bootstrap.yml

# Phase 2 — harden
ansible-playbook -i inventory/vagrant.yml site.yml

# End-to-end test
ansible-playbook -i inventory/vagrant.yml tests/test-harden-e2e.yml
```

### Remote / Proxmox VM

Point `inventory/proxmox.yml` and `inventory/proxmox-bootstrap.yml` at the
target VM's IP (`ansible_host`), then choose the bootstrap variant that matches
your image.

**Variant A — fresh install with root password auth enabled:**

```bash
# Phase 1 — bootstrap as root with password
ansible-playbook -i inventory/proxmox-bootstrap.yml playbooks/bootstrap.yml \
  -e ansible_user=root --ask-pass --ask-become-pass
```

**Variant B — image with password auth disabled** (e.g. a cloud-init image
where only the pre-installed user's key is present):

```bash
# Install the deploy pubkey for the sudo user that the image provides, then:
ansible-playbook -i inventory/proxmox-bootstrap.yml playbooks/bootstrap.yml \
  -e ansible_user=<cloud-user>
```

**Phase 2 and testing (both variants):**

```bash
# Phase 2 — harden (connects as the ansible deploy user)
ansible-playbook -i inventory/proxmox.yml --ask-vault-pass site.yml

# End-to-end test
ansible-playbook -i inventory/proxmox.yml tests/test-harden-e2e.yml
```

> **Note:** all inventories set `ansible_ssh_common_args: "-F /dev/null ..."`.
> The `-F /dev/null` flag is harmless on most systems and required on some
> control hosts (e.g. NixOS with a libvirt ssh_config drop-in that OpenSSH
> rejects due to permissions); it makes ssh ignore the system ssh_config
> entirely.

---

## 2FA enrollment

After the harden playbook completes, human operators must enroll before
`ssh_2fa_totp_nullok` is set to `false`.

### TOTP (google-authenticator)

Log in to the managed host as the human admin user and run:

```bash
google-authenticator --time-based --disallow-reuse --force --rate-limit=3 \
  --rate-time=30 --window-size=3
```

Save the secret and scratch codes in a password manager. The command creates
`~/.google_authenticator`. Repeat for every human account that uses SSH.

### FIDO2 / hardware key (sk-ed25519)

```bash
# On the operator workstation:
ssh-keygen -t ed25519-sk -O verify-required -f ~/.ssh/id_ed25519_sk -C "fido2-$(hostname)"

# Add the public key to the managed host's authorized_keys for the user.
# Then add the user to the fido2users group on the managed host:
sudo usermod -aG fido2users <username>
```

FIDO2 users authenticate with key touch plus PIN and only sk-ed25519 keys are
accepted for them; TOTP is not required for accounts
in the `fido2users` group.

### Lock down nullok after enrollment

Once every human operator has enrolled (TOTP file present or FIDO2 key + group
membership in place), disable the TOTP grace mode:

```yaml
# inventory/group_vars/all/main.yml
ssh_2fa_totp_nullok: false
```

```bash
ansible-playbook -i inventory/proxmox.yml --ask-vault-pass site.yml
```

The `ansible` deploy account is always exempt from 2FA — it authenticates by
key only.

---

## Lockout recovery

If a run leaves the host unreachable before the dead-man's-switch fires, or
after `ssh_2fa_totp_nullok` is set to `false` before enrollment is complete,
use the **Proxmox / hypervisor console** to log in directly.

From the console:

```bash
# Remove the SSH hardening drop-ins and reload sshd
rm -f /etc/ssh/sshd_config.d/10-hardening.conf /etc/ssh/sshd_config.d/20-2fa.conf
systemctl reload ssh
```

The firewall is base-debian's (`inet base_filter`). If it is blocking you,
re-run base-debian to restore the intended ruleset. `nft flush ruleset` is a
last resort only — it clears *all* firewalling (base_filter, crowdsec, and
fail2ban bans); re-run base-debian afterward.

**Dead-man's-switch:** when `harden_safe_rollback: true` (the default), the
playbook arms a transient systemd timer at run start. If Ansible does not
disarm it upon successful completion, the timer fires after
`harden_rollback_minutes` (default: 20) minutes and automatically restores
the pre-run SSH drop-ins state. This gives a 20-minute
automatic recovery window without requiring console access.
