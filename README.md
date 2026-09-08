# Ansible Role — Accounting

Manages Linux users, groups, sudoers, and per-user shell customizations across your fleet. Each user and group is defined in its own YAML file and selectively applied to hosts via a **`targets`** list.

## How Targeting Works

Every user and group definition includes a `targets` list. An entry is applied to a host when **any** target matches:

| Target value | Matches when… |
|---|---|
| `all` | Always — every host in the play |
| An **Ansible group name** | The host belongs to that inventory group (e.g. `webservers`) |
| A **hostname** | The host's `inventory_hostname` matches exactly (e.g. `db-primary`) |

```yaml
# Applied to hosts in 'webservers' or 'devboxes'
username: jdoe
targets: [webservers, devboxes]

# Applied to every host
username: deploy
targets: [all]

# Applied only to host 'db-primary'
username: dbadmin
targets: [db-primary]
```

## Requirements

- Ansible ≥ 2.9
- `ansible.posix` collection (for `authorized_key`)
- Target hosts running a Debian/RHEL-based Linux distribution

## Quick Start

```bash
# 1. Copy the example files and customise
cp roles/accounting/vars/groups/example.yaml.dist roles/accounting/vars/groups/mygroup.yaml
cp roles/accounting/vars/users/example.yaml.dist  roles/accounting/vars/users/myuser.yaml

# 2. Edit definitions
vim roles/accounting/vars/groups/mygroup.yaml
vim roles/accounting/vars/users/myuser.yaml

# 3. Set your inventory
vim inventory.ini

# 4. Run the playbook
ansible-playbook -i inventory.ini site.yaml
```

## Role Variables

### Defaults (`defaults/main.yaml`)

| Variable | Default | Description |
|---|---|---|
| `accounting_groups_var_dir` | `<role>/vars/groups` | Directory of per-group YAML files |
| `accounting_users_var_dir` | `<role>/vars/users` | Directory of per-user YAML files |
| `accounting_default_shell` | `/bin/bash` | Default login shell for new users |
| `accounting_home_base` | `/home` | Base path for home directories |
| `accounting_create_home` | `true` | Whether to create home directories |
| `accounting_default_groups` | `[]` | Supplementary groups added to **every** user |
| `accounting_remove_absent` | `true` | Remove users marked `state: absent` |
| `accounting_remove_home` | `false` | Delete home directory when removing a user |
| `accounting_default_options` | `{expires: -1, ...}` | Default password expiration options |

## Definitions

### Group Fields

Each file in `vars/groups/` is a flat YAML document defining one group:

| Field | Required | Description |
|---|---|---|
| `group_name` | ✅ | Group name |
| `gid` | | Numeric GID |
| `state` | | `present` *(default)* / `absent` |
| `system` | | Create as a system group |
| `targets` | ✅ | Where to apply — list of group names, hostnames, or `all` |

```yaml
# vars/groups/developers.yaml
group_name: developers
gid: 2000
targets:
  - webservers
  - devboxes
```

### User Fields

Each file in `vars/users/` is a flat YAML document defining one user:

| Field | Required | Description |
|---|---|---|
| `username` | ✅ | Login name |
| `uid` | | Numeric UID |
| `gid` | | Numeric GID (used for primary group creation) |
| `group` | | Primary group (auto-created if it doesn't exist) |
| `groups` | | Supplementary groups (enforced — groups not listed are removed; only existing groups are applied) |
| `shell` | | Login shell (falls back to `accounting_default_shell`) |
| `home` | | Home directory path (auto-generated from `accounting_home_base`) |
| `comment` | | GECOS / full name |
| `password` | | Hashed password. If omitted, a secure 32-character random password is generated and hashed automatically. |
| `options` | | Dict for account and password expiration (e.g. `expires`, `password_expire_max`) |
| `system` | | Create as a system account |
| `state` | | `present` *(default)* / `absent` |
| `manage_account` | | Set to `false` to skip the `user` module (customizations still apply — useful for root) |
| `targets` | ✅ | Where to apply — list of group names, hostnames, or `all` |

#### Access & Security

| Field | Description |
|---|---|
| `sudo` | `true` — shorthand for `sudoers: { nopasswd: false }` if `sudoers` is not defined |
| `sudoers` | Dict for fine-grained sudoers config (see [Sudoers](#sudoers) below) |
| `ssh_keys` | List of SSH public key strings — deployed with `exclusive: true` (keys not listed are removed). If omitted or empty, the `authorized_keys` file is removed. |

#### Shell Customization

| Field | Description |
|---|---|
| `aliases` | Dict of `name: command` pairs → written to `~/.bash_aliases` |
| `bashrc_block` | Multi-line string appended to `~/.bashrc` |
| `dotfiles` | List of `{src, dest, mode}` files copied into the home directory |

#### Provisioning

| Field | Description |
|---|---|
| `provision` | List of task file paths (relative to `tasks/`) to include for this user |

Additional custom fields (e.g. `omb_theme`, `omb_plugins`, `packages`, `dotfiles_repo`) can be added and will be accessible inside provision tasks via the `_user` variable.

<details>
<summary>Full example — <code>vars/users/jdoe.yaml</code></summary>

```yaml
username: jdoe
uid: 2000
comment: "John Doe"
group: developers
groups:
  - docker
sudo: true
sudoers:
  nopasswd: true
shell: /bin/bash
ssh_keys:
  - "ssh-ed25519 AAAAC3... jdoe@laptop"

aliases:
  ll: "ls -lah --color=auto"
  gs: "git status"
  gp: "git pull"
  dc: "docker compose"
  k: "kubectl"

bashrc_block: |
  export EDITOR=vim
  export PATH=$HOME/.local/bin:$PATH
  [ -f ~/.bash_aliases ] && . ~/.bash_aliases

provision:
  - provision/oh-my-bash.yaml
omb_theme: powerline
omb_plugins:
  - git
  - bashmarks
  - npm

targets:
  - webservers
  - devboxes
```

</details>

<details>
<summary>Root account example — <code>vars/users/root.yaml</code></summary>

```yaml
# manage_account: false skips the user module (avoids usermod errors on root)
# but still applies customizations like bashrc, aliases, and provision tasks
username: root
home: /root
manage_account: false
provision:
  - provision/oh-my-bash.yaml
omb_theme: binaryanomaly
bashrc_block: |
  export EDITOR=nano
aliases:
  ll: "ls -lah --color=auto"
targets:
  - all
```

</details>

## Sudoers

Users with a `sudoers` dict get a validated file dropped into `/etc/sudoers.d/`. Two modes are supported:

**Full sudo (with or without password):**

```yaml
sudoers:
  nopasswd: true          # optional — defaults to false
  runas: ALL              # optional — defaults to ALL
```

**Command-restricted sudo:**

```yaml
sudoers:
  commands:
    - /usr/bin/systemctl restart *
    - /usr/bin/journalctl
  nopasswd: true
  runas: ALL
```

Sudoers files are automatically removed when a user is set to `state: absent`.

## Per-User Provisioning

The `provision` field accepts a list of task files that are `include_tasks`'d with the full user dict available as `_user`. This lets you attach arbitrary automation to individual users.

Two built-in provision files are included:

### `provision/oh-my-bash.yaml`

Installs [oh-my-bash](https://github.com/ohmybash/oh-my-bash), sets theme and plugins.

| User field | Description |
|---|---|
| `omb_theme` | oh-my-bash theme name (default: `powerline`) |
| `omb_plugins` | List of oh-my-bash plugins |

### `provision/dev-tools.yaml`

Installs packages and clones a dotfiles repository.

| User field | Description |
|---|---|
| `packages` | List of system packages to install |
| `dotfiles_repo` | Git URL cloned to `~/.dotfiles` |
| `dotfiles_branch` | Branch to check out (default: `main`) |

## File Structure

```
roles/accounting/
├── defaults/
│   └── main.yaml                ← role defaults
├── templates/
│   └── sudoers.j2               ← per-user sudoers template
├── tasks/
│   ├── main.yaml                ← orchestrator (loads vars, calls sub-tasks)
│   ├── groups.yaml              ← create / remove groups
│   ├── users.yaml               ← create / remove users, SSH keys, dotfiles
│   ├── sudoers.yaml             ← deploy / remove sudoers files
│   └── provision/
│       ├── oh-my-bash.yaml      ← oh-my-bash installer
│       └── dev-tools.yaml       ← package + dotfiles provisioner
└── vars/
    ├── groups/
    │   ├── example.yaml.dist    ← template — copy & rename to add a group
    │   ├── dbadmins.yaml
    │   ├── developers.yaml
    │   ├── docker.yaml
    │   └── ops.yaml
    └── users/
        ├── example.yaml.dist    ← template — copy & rename to add a user
        ├── dbadmin.yaml
        ├── deploy.yaml
        ├── jdoe.yaml
        └── olduser.yaml
```

## Extra Variables (Runtime Options)

You can use the `-e` flag to pass extra variables at runtime to modify playbook execution:

| Variable | Description | Example |
|---|---|---|
| `username` | Filter execution to a single user. | `-e username=bob` |
| `groupname` | Filter execution to a single group. | `-e groupname=developers` |
| `update_password` | Force password updates. Defaults to `false` (only updates on creation). Set to `true` to force a password reset/update. | `-e update_password=true` |

```bash
# Run the role only for the user 'bob' and force password update
ansible-playbook playbook.yml -e username=bob -e update_password=true
```

## Tags

Each task group can be selectively run or skipped:

| Tag | Controls |
|---|---|
| `groups` | Group creation / removal |
| `users` | User accounts, SSH keys, shell customizations, provisioning |
| `sudo` | Sudo package install, sudoers file deployment / removal |

```bash
# Only manage sudoers
ansible-playbook -i inventory.ini site.yaml --tags sudo

# Skip sudoers entirely
ansible-playbook -i inventory.ini site.yaml --skip-tags sudo

# Only users and groups, no sudoers
ansible-playbook -i inventory.ini site.yaml --tags groups,users
```

> **Note:** Variable loading tasks are tagged `always` and will run regardless of tag selection, ensuring definitions are available to whichever task group you choose.

## Task Execution Order

1. **Discover & load** group definitions from `vars/groups/`
2. **Discover & load** user definitions from `vars/users/`
3. **Groups** — filter by targets, then create or remove
4. **Users** — filter by targets, then:
   - Create / manage user accounts
   - Deploy SSH authorized keys
   - Write `~/.bashrc` blocks and `~/.bash_aliases`
   - Copy dotfiles
   - Run per-user provision tasks
   - Remove absent users
5. **Sudoers** — deploy or remove `/etc/sudoers.d/` files

## License

See the project-level [LICENSE](../../LICENSE) if applicable.
