# Ansible Role: ufw

Declarative management of UFW (Uncomplicated Firewall) rules on Ubuntu.

Rules are declared as data — a dictionary of services and their fields —
rather than as individual `community.general.ufw` tasks. The role expands
that data into UFW rules and keeps the host in sync with it: missing or
modified rules are restored, and rules that are not declared (including
rules added by hand) are removed.

NAT (masquerading, port forwarding) is not supported yet. Rules for
forwarded traffic (`route: true`) are supported, but NAT rules added by
hand to `/etc/ufw/before.rules` are lost on every rebuild (see
[Caveats](#caveats)).

## Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Example playbook](#example-playbook)
- [Role variables](#role-variables)
- [Rule fields](#rule-fields)
- [Rule expansion](#rule-expansion)
- [Rule order](#rule-order)
- [Layering rules across the inventory](#layering-rules-across-the-inventory)
- [How reconciliation works](#how-reconciliation-works)
- [Caveats](#caveats)
- [Testing](#testing)
- [License](#license)
- [Author](#author)

## Features

- Rules defined as data; every `community.general.ufw` rule parameter is
  supported.
- Rule layers for all hosts, inventory groups and individual hosts, merged
  per service.
- List-valued fields expand into one rule per combination.
- Full reconciliation: undeclared rules are removed, changed rules are
  restored.
- Idempotent; a converged host reports no changes.
- Check mode (`--check`) reports drift without modifying the host.

## Requirements

- Ansible (ansible-core) 2.16 or later.
- The [`community.general`](https://galaxy.ansible.com/ui/repo/published/community/general/)
  collection. It is included in the full `ansible` package; with
  `ansible-core` alone, install it with
  `ansible-galaxy collection install community.general`.
- On the managed hosts: the `ufw` package installed (the role does not
  install it), and privilege escalation (`become: true`).

Supported and tested platforms:

| Platform         | Codename |
|------------------|----------|
| Ubuntu 24.04 LTS | noble    |
| Ubuntu 26.04 LTS | resolute |

## Installation

The role is published on
[Ansible Galaxy](https://galaxy.ansible.com/ui/standalone/roles/Maksym-Ototiuk/ufw/)
as `Maksym-Ototiuk.ufw`. Pin it to a release in the `requirements.yml` of
your Ansible project:

```yaml
# requirements.yml
roles:
  - name: Maksym-Ototiuk.ufw
    version: v1.0.1

collections:
  - name: community.general
```

```bash
ansible-galaxy install -r requirements.yml
```

Or install it directly:

```bash
ansible-galaxy role install Maksym-Ototiuk.ufw,v1.0.1
```

The role is installed into the default roles path (`~/.ansible/roles`);
add `-p roles/` to install it into the project instead. Reference it in
playbooks as `Maksym-Ototiuk.ufw`.

### Installing under a local name

To use a shorter name in your playbooks, set `name` in
`requirements.yml`. The role is downloaded from Galaxy as usual, but
installed under the name you choose:

```yaml
# requirements.yml
roles:
  - src: Maksym-Ototiuk.ufw
    name: ufw
    version: v1.0.1
```

```bash
ansible-galaxy install -r requirements.yml
```

The role is then available as `ufw`:

```yaml
- name: Configure firewall
  hosts: all
  become: true
  roles:
    - ufw
```

To upgrade, change `version` and run
`ansible-galaxy install -r requirements.yml --force`.

### Installing from GitHub

```yaml
# requirements.yml
roles:
  - src: https://github.com/Maksym-Ototiuk/ansible-role-ufw.git
    scm: git
    name: ufw
    version: v1.0.1
```

Available versions are listed in [CHANGELOG.md](CHANGELOG.md). The role
follows [Semantic Versioning](https://semver.org/).

## Example playbook

```yaml
- name: Configure firewall
  hosts: all
  become: true
  roles:
    - Maksym-Ototiuk.ufw
```

The rules come from inventory variables; see
[Layering rules across the inventory](#layering-rules-across-the-inventory)
for a complete example.

## Role variables

All variables default to `{}`.

| Variable     | Typical location            | Purpose                                                    |
|--------------|-----------------------------|------------------------------------------------------------|
| `ufw_common` | `group_vars/all.yml`        | Rules for every host.                                      |
| `ufw_group`  | `group_vars/<group>.yml`    | Rules for an inventory group.                              |
| `ufw_host`   | `host_vars/<host>.yml`      | Rules for a single host.                                   |
| `ufw`        | anywhere                    | Complete rule set for the host; if non-empty, the three layers above are ignored. |

Each variable is a dictionary keyed by a **service name**. The service
name is only a label: it appears in task output and is appended to the
rule comment. The value is a dictionary of rule fields:

```yaml
ufw_common:
  ssh:
    port: 22
    proto: tcp
    src: 10.0.0.0/8
    comment: SSH from the management network
```

If all four variables are empty for a host, the role does nothing on that
host.

## Rule fields

Field names are the `community.general.ufw` parameter names or their
documented aliases. All fields are optional, and every field accepts a
single value or a list (see [Rule expansion](#rule-expansion)).

| Field                | Description                                                                                 |
|----------------------|---------------------------------------------------------------------------------------------|
| `rule`               | `allow`, `deny`, `reject` or `limit`. Default: `allow`.                                     |
| `proto`              | `tcp`, `udp`, `ipv6`, `esp`, `ah`, `gre`, `igmp` or `any`. Cannot be combined with `name`.  |
| `src`                | Source address or network. Default: `any`.                                                  |
| `from_port`          | Source port.                                                                                |
| `dest`               | Destination address or network. Default: `any`.                                             |
| `port`               | Destination port or port range (`"60000:61000"`).                                           |
| `direction`          | `in` or `out`. Required when `interface` is set.                                            |
| `interface`          | Interface the rule applies to, in the given `direction`.                                    |
| `interface_in`       | Incoming interface. Cannot be combined with `direction`.                                    |
| `interface_out`      | Outgoing interface. Cannot be combined with `direction`.                                    |
| `route`              | `true` for a rule on routed (forwarded) traffic.                                            |
| `name`               | Application profile from `/etc/ufw/applications.d`, for example `OpenSSH`.                  |
| `log`                | `true` to log new connections that match the rule.                                          |
| `insert`             | Position to insert the rule at. See [Rule order](#rule-order).                              |
| `insert_relative_to` | `zero` (default), `first-ipv4`, `last-ipv4`, `first-ipv6` or `last-ipv6`. See [Rule order](#rule-order). |
| `comment`            | Rule comment. The service name is appended: `SSH access (ssh)`. Must not contain `'`.       |

Fields that are not set are not passed to the module, so the module
defaults apply.

## Rule expansion

When fields are lists, the role creates one rule for every combination of
their values. The number of rules for a service is the **product** of the
list lengths, where a single value counts as 1:

| Fields                                         | Rules |
|------------------------------------------------|------:|
| `port: [80, 443]`                              | 2     |
| `port: [80, 443]`, `src: [a, b, c]`            | 6     |
| `port: [80, 443]`, `src: [a, b, c]`, `proto: [tcp, udp]` | 12 |

The count grows quickly. A variable that resolves to a long list — for
example `src: "{{ office_networks }}"` with 20 entries — combined with a
few ports can produce hundreds of rules from one service. Use lists only
where every combination is wanted, and keep fields such as `rule`,
`direction`, `log` and `route` scalar.

## Rule order

UFW evaluates rules in order and applies the first match. When the role
rebuilds the rule set, it applies rules in declaration order:

- services from `ufw_common` first, then new services from `ufw_group`,
  then new services from `ufw_host`;
- within a layer, services in the order they are written;
- within a service, expanded combinations in field order.

A service redefined in a later layer keeps the position of its first
definition.

To place a rule above rules declared earlier — for example a host-specific
`deny` above a common `allow` — use `insert` with the default
`insert_relative_to: zero`. Two restrictions apply:

- the position must not be greater than the number of rules applied
  before it, or the rebuild fails;
- other `insert_relative_to` values have no effect during a rebuild, and
  the rule is appended: UFW is disabled while rules are applied, so the
  module cannot determine the current rule positions.

The drift check does not detect rules that are present but in a different
position.

## Layering rules across the inventory

The effective rule set for a host is computed as follows:

```yaml
ufw_effective_rules: >-
  {{ ufw if (ufw | length > 0) else
     ({} | combine(ufw_common, recursive=True)
         | combine(ufw_group,  recursive=True)
         | combine(ufw_host,   recursive=True)) }}
```

Because the merge is recursive, a service defined in more than one layer
has its **fields** merged: a later layer can override one field (for
example `src`) and inherit the rest. A list value replaces the earlier
list; lists are not concatenated.

Each layer is an ordinary Ansible variable, so normal variable precedence
applies within it. For example, if a host is a member of two groups that
both define `ufw_group`, only one of the two definitions is used. Use
`ufw_group` for one group per host, or give the groups distinct services
in `ufw_host` or `ufw_common`.

### Example

```yaml
# group_vars/all.yml
ufw_common:
  ssh:
    port: 22
    proto: tcp
    src: 10.0.0.0/8
    comment: SSH from the management network
```

```yaml
# group_vars/webservers.yml
ufw_group:
  web:
    port: [80, 443]
    proto: tcp
    comment: Web traffic
```

```yaml
# host_vars/web01.example.com.yml
ufw_host:
  ssh:
    src: 192.0.2.10        # replaces src; port, proto and comment are inherited
  metrics:
    port: 9100
    proto: tcp
    src: 192.0.2.20
    comment: Prometheus node exporter
```

Resulting rules on `web01.example.com`:

```text
ufw allow from 192.0.2.10 to any port 22 proto tcp comment 'SSH from the management network (ssh)'
ufw allow 80/tcp comment 'Web traffic (web)'
ufw allow 443/tcp comment 'Web traffic (web)'
ufw allow from 192.0.2.20 to any port 9100 proto tcp comment 'Prometheus node exporter (metrics)'
```

Other hosts in `webservers` get the `ssh` rule from `ufw_common`
unchanged, plus the `web` rules.

To give one host a completely separate rule set, set `ufw` for it:

```yaml
# host_vars/bastion.example.com.yml
ufw:
  ssh:
    port: 22
    proto: tcp
    comment: Public SSH
```

## How reconciliation works

On every run the role first checks the host without changing it. A
rebuild is required if any of these is true:

1. UFW is not enabled.
2. A declared rule is missing or differs from the declaration. Each
   expanded rule is checked with `ufw --dry-run`.
3. The number of rules on the host (`ufw show added`) differs from the
   number of declared rules. This catches rules that were added outside
   Ansible.

If a rebuild is required, the role runs `ufw reset`, applies all declared
rules and enables UFW. Otherwise it makes no changes.

In check mode the role reports whether a rebuild would happen; the host
is not modified.

## Caveats

Read this section before running the role against production hosts.

- **The firewall is open during a rebuild.** `ufw reset` disables UFW, and
  it stays disabled until all rules are applied. The duration depends on
  the number of rules.
- **A failed rebuild leaves UFW disabled.** If a rule fails to apply after
  the reset, the role stops with an error stating that UFW is disabled.
  Fix the cause and run the role again. Most errors (invalid addresses,
  unknown application profiles) are caught by the dry run, before the
  reset.
- **Keep a rule for your SSH access.** The default incoming policy is
  `deny`. If the declared rules do not allow SSH from the Ansible
  controller, new SSH connections are blocked once UFW is enabled.
- **`/etc/ufw/*.rules` are restored to package defaults** on every
  rebuild, so manual changes to `before.rules` and `after.rules` (for
  example NAT rules) are lost. `ufw reset` keeps the previous files as
  timestamped backups in `/etc/ufw/`; these accumulate with each rebuild.
- **Default policies and logging are not managed.** Configure them
  separately (for example with `community.general.ufw` using `default`
  and `logging`).
- **Declared rules must be distinct.** Two declared rules that differ only
  in their comment become one UFW rule. The role detects this after a
  rebuild and fails with an explanation; otherwise every run would rebuild
  the firewall again.

## Testing

The tests use [Molecule](https://ansible.readthedocs.io/projects/molecule/)
with Docker. Each supported Ubuntu release runs in a privileged container
(UFW needs iptables); every container has its own network namespace, so
the firewall of the machine running the tests is not changed.

The test scenario (`molecule/default`) checks:

- applying rules from all layers, covering every rule field, and comparing
  the result with the expected `ufw show added` output;
- idempotence and check mode on a converged host;
- detection and repair of undeclared, deleted and modified rules and of a
  disabled firewall, with check mode reporting drift without changing the
  host;
- the `ufw` override, and an empty rule set leaving UFW untouched;
- rejection of invalid comments, detection of duplicate rules, and the
  error raised when a rebuild fails.

Run the tests locally (requires Docker and Python 3.10 or later):

```bash
python3 -m venv ~/.venvs/molecule
~/.venvs/molecule/bin/pip install ansible-core ansible-lint yamllint molecule "molecule-plugins[docker]"
source ~/.venvs/molecule/bin/activate

yamllint .
ansible-lint
molecule test
```

GitHub Actions runs the same checks on every push and pull request, with
the latest ansible-core and with ansible-core 2.16.

## License

MIT. See [LICENSE](LICENSE).

## Author

Maksym Ototiuk — contact_me@masik.slmail.me
