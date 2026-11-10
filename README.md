# Automated Ubuntu Security Compliance Auditing and Enforcement with Ansible

## 1. Project purpose

This project uses Ansible to audit an Ubuntu Linux machine against a small security baseline and provides a separate, controlled remediation workflow.

The project is designed for a learning environment with one Ubuntu VM. The inventory uses Ansible's local connection, so the Ansible controller and managed target are the same Ubuntu VM. No cloud server or additional VM is required.

The project demonstrates:
- Ansible inventory and configuration.
- Reusable roles and task inclusion.
- Variables and group variables.
- Privilege escalation with `become`.
- Audit and remediation separation.
- Ansible Vault basics.
- Idempotent configuration modules.
- JSON and HTML compliance reports.

This is a learning baseline, not a certified CIS benchmark or a production security product. Review the checks and adapt them to the tutor's exact rubric and Ubuntu version.

## 2. Project structure

```text
ansible-compliance/
├── ansible.cfg
├── requirements.yml
├── README.md
├── inventory/
│   ├── hosts.ini
│   └── group_vars/all/
│       ├── vars.yml
│       └── vault.yml.example
├── playbooks/
│   ├── audit.yml
│   ├── remediate.yml
│   └── vault-test.yml
├── roles/
│   ├── firewall_compliance/
│   ├── user_compliance/
│   ├── file_permissions/
│   ├── service_compliance/
│   ├── update_compliance/
│   ├── logging_compliance/
│   ├── network_compliance/
│   ├── ssh_compliance/
│   └── compliance_report/
└── reports/
```

Each compliance role has:
- `defaults/main.yml`: role defaults.
- `tasks/main.yml`: entry point that chooses audit or remediation.
- `tasks/audit.yml`: inspection tasks.
- `tasks/remediate.yml`: tasks that apply approved changes.
- `handlers/main.yml`: reserved for service reload/restart handlers when needed.

## 3. Controls included

| Control ID | Area | Audit behaviour | Remediation behaviour |
|---|---|---|---|
| FW-01 | Firewall | Checks UFW installation, active state and default incoming policy | Installs UFW, allows configured SSH port, sets policies and enables UFW only with explicit approval |
| USER-01 | Users | Checks empty local password fields and sudo/wheel group members | Reports findings; user/password changes remain manual |
| FILE-01 | File permissions | Checks configured modes for `/etc/passwd` and `/etc/shadow` | Applies configured modes |
| SVC-01 | Services | Checks telnet, rsh and rlogin service states | Stops/disables these services if present |
| UPD-01 | Updates | Lists packages reported as upgradeable by apt | Performs a safe apt upgrade |
| LOG-01 | Logging | Checks whether journald or rsyslog is running | Installs and starts rsyslog |
| NET-01 | Network | Checks selected IPv4 forwarding and redirect sysctl values | Persists configured values in `/etc/sysctl.d/` |
| SSH-01 | SSH | Produces a manual review item for effective SSH settings | Does not automatically change SSH configuration |

### Status definitions

- `COMPLIANT`: the check matches the configured expectation.
- `NON_COMPLIANT`: the check found a mismatch.
- `REVIEW`: a human must inspect the evidence or decide what action is appropriate.

The update check uses `REVIEW` when upgradeable packages are listed because `apt list --upgradable` does not, by itself, prove that every listed package is a security update.

## 4. Safety decisions

### Local execution

`inventory/hosts.ini` contains:

```ini
[compliance_targets]
ubuntu-vm ansible_connection=local
```

Ansible runs locally on the Ubuntu VM. This is independent of the Windows-to-Ubuntu Remote-SSH connection used by VS Code.

### Audit versus remediation

`playbooks/audit.yml` selects audit tasks. Audit tasks are designed to inspect the machine. They may use privileged read commands, but should not intentionally change security settings.

`playbooks/remediate.yml` selects remediation tasks. It can change system configuration and must only be run after reviewing the audit results.

### SSH

SSH remediation is intentionally manual. Do not disable password authentication or change root login settings until you have confirmed an alternate console or recovery path and verified the actual SSH configuration.

### Firewall

Firewall remediation is blocked by default. Before enabling it:
1. Confirm the actual SSH listening port.
2. Confirm the connection path to the VM.
3. Update `compliance_ssh_port` in `inventory/group_vars/all/vars.yml` if necessary.
4. Set `compliance_allow_firewall_remediation: true` only after reviewing the risk.

The sample port is TCP/22. Do not assume that port is correct for every machine.

### User accounts

The project reports empty password fields and supplementary membership in the `sudo` and `wheel` groups. It does not automatically delete accounts, change passwords, or rewrite sudoers configuration.

## 5. Prerequisites

- Ubuntu Linux VM.
- Python 3 installed.
- Ansible Core installed.
- A user with sudo privileges.
- Internet access to install Ansible collections and retrieve apt package metadata when remediation is selected.

The project was written for Ubuntu with systemd and apt. Some service names and package behaviour may differ on other Linux distributions.

## 6. Installation and preparation

Open a terminal in the extracted project directory.

### Step 1: Confirm the current directory

```bash
pwd
ls
```

You should be in the project root, where `ansible.cfg`, `requirements.yml`, `inventory`, `playbooks` and `roles` are located.

### Step 2: Check Ansible

```bash
ansible --version
```

### Step 3: Install required collections

```bash
ansible-galaxy collection install -r requirements.yml
```

The project uses:
- `community.general` for the UFW module.
- `ansible.posix` for the sysctl module.

### Step 4: Check the inventory

```bash
ansible-inventory --graph
ansible-inventory --host ubuntu-vm
```

### Step 5: Test local connectivity

```bash
ansible compliance_targets -m ping
```

A successful result includes `ping: pong`.

## 7. Ansible Vault setup (optional demonstration)

The compliance audit does not require a Vault variable. Vault is included to demonstrate how Ansible encrypts sensitive variables.

Copy the example:

```bash
cp inventory/group_vars/all/vault.yml.example inventory/group_vars/all/vault.yml
```

Edit the file:

```bash
nano inventory/group_vars/all/vault.yml
```

Encrypt it:

```bash
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

View it:

```bash
ansible-vault view inventory/group_vars/all/vault.yml
```

Edit it:

```bash
ansible-vault edit inventory/group_vars/all/vault.yml
```

Test variable loading:

```bash
ansible-playbook playbooks/vault-test.yml --ask-vault-pass
```

The Vault password is separate from the Ubuntu sudo password. Never put real passwords or API keys in the example file. Do not commit the encrypted Vault file to a public repository unless the project policy explicitly permits it; never commit an unencrypted secrets file.

If you do not create `vault.yml`, skip the Vault test and do not add `--ask-vault-pass` to the audit command.

## 8. Run the audit

From the project root:

```bash
ansible-playbook playbooks/audit.yml --ask-become-pass
```

Enter the Ubuntu account password at the `BECOME password` prompt.

This playbook:
1. Targets `compliance_targets`.
2. Gathers system facts.
3. Starts with an empty result list.
4. Runs each role's audit tasks.
5. Collects each control result.
6. Generates JSON and HTML reports.
7. Displays a short summary.

If you created an encrypted Vault file, add `--ask-vault-pass`:

```bash
ansible-playbook playbooks/audit.yml --ask-vault-pass --ask-become-pass
```

`--ask-vault-pass` decrypts Vault variables. `--ask-become-pass` supplies the sudo password. They are different prompts for different purposes.

## 9. Review the reports

The report role writes:

```text
reports/
├── ubuntu-vm-compliance-report.json
└── ubuntu-vm-compliance-report.html
```

The JSON report is suitable for scripts and further processing. The HTML report is easier for a person to read and can be opened in a browser.

Review every `NON_COMPLIANT` and `REVIEW` result before taking action.

## 10. Run remediation

Remediation can change the Ubuntu machine. Review the audit report first and take a VM snapshot or other suitable backup before making changes.

Run:

```bash
ansible-playbook playbooks/remediate.yml --ask-become-pass
```

If an encrypted Vault file exists, add `--ask-vault-pass`.

Firewall remediation will stop at an explicit approval check unless `compliance_allow_firewall_remediation` is set to `true`. Confirm the actual SSH port before changing that setting.

User and SSH changes are intentionally left for a human administrator. Other remediation tasks include file modes, prohibited services, package upgrades, rsyslog and network sysctl settings.

After remediation, run the audit again:

```bash
ansible-playbook playbooks/audit.yml --ask-become-pass
```

Compare the new reports with the previous results.

## 11. Useful Ansible commands

```bash
# Show Ansible version
ansible --version

# Show inventory groups
ansible-inventory --graph

# Show variables for a host
ansible-inventory --host ubuntu-vm

# Test connectivity
ansible compliance_targets -m ping

# Check playbook syntax without executing tasks
ansible-playbook playbooks/audit.yml --syntax-check

# Preview task list
ansible-playbook playbooks/audit.yml --list-tasks

# Run audit with sudo password prompt
ansible-playbook playbooks/audit.yml --ask-become-pass

# Run remediation with sudo password prompt
ansible-playbook playbooks/remediate.yml --ask-become-pass

# Encrypt a Vault file
ansible-vault encrypt inventory/group_vars/all/vault.yml

# Edit an encrypted Vault file
ansible-vault edit inventory/group_vars/all/vault.yml
```

## 12. Submission checklist

Include the following in the assignment submission:
1. Complete project directory and README.
2. Ansible version output.
3. Inventory graph and successful ping output.
4. Successful syntax check.
5. Audit playbook output.
6. Generated JSON and HTML reports.
7. Explanation of each compliance area.
8. Explanation of why audit and remediation are separate.
9. Explanation of how Vault protects sensitive variables.
10. Evidence that SSH and firewall changes were handled cautiously.
