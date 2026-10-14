# ansible-role-jumpserver-host

Registers a Linux host as an asset in JumpServer through its REST API
(`/api/v1/assets/hosts/`). API calls run on the control node (`delegate_to: localhost`).

Behaviour:

- `present`: creates the asset if no asset with the exact name exists; otherwise patches
  `address`, `is_active` and `comment` when they differ.
- `absent`: deletes the asset if it exists.
- Platform, nodes, zone and account templates are given by name/path and resolved to ids via the API.
- Platform, nodes, zone, protocols, labels and accounts are applied **only on creation**.
- API tokens never appear in logs (`no_log`).

Tested with Ansible 8.6 (ansible-core 2.15) and a JumpServer v3+/v4 API.

## Variables

### Connection

| Variable | Default | Description |
|---|---|---|
| `jumpserver_host` | `""` (required) | JumpServer hostname or full URL. `https://` is added if there is no scheme. |
| `jumpserver_token` | `""` (required) | Private token, sent as `Authorization: Token ...`. |
| `jumpserver_validate_certs` | `true` | Validate the TLS certificate. |
| `jumpserver_api_timeout` | `30` | API request timeout in seconds. |

### Asset

| Variable | Default | Description |
|---|---|---|
| `jumpserver_asset_state` | `present` | `present` or `absent`. |
| `jumpserver_asset_name` | `{{ inventory_hostname }}` | Asset name in JumpServer (exact-match lookup for idempotency). |
| `jumpserver_asset_address` | `{{ ansible_host \| default(inventory_hostname) }}` | IP or hostname JumpServer connects to. |
| `jumpserver_asset_platform` | `Linux` | Platform name or numeric id. |
| `jumpserver_asset_protocols` | `[{name: ssh, port: 22}]` | List of `{name, port}`. |
| `jumpserver_asset_nodes` | `[]` | Nodes by full path (e.g. `/DEFAULT/test-node1/sub-node1`) or UUID. Empty = default node. Unknown path fails the play. |
| `jumpserver_asset_zone` | `""` | Zone (the one with the gateway) by name or UUID, applied on creation. Empty = no zone. Unknown name fails the play. |
| `jumpserver_asset_labels` | `[]` | Labels as accepted by the API. |
| `jumpserver_asset_comment` | `Managed by Ansible` | Asset comment. |
| `jumpserver_asset_is_active` | `true` | Whether the asset is enabled. |

### Accounts

| Variable | Default | Description |
|---|---|---|
| `jumpserver_asset_account_templates` | `[]` | Account template names (e.g. `[root]`) attached on creation. Unknown name fails the play. |
| `jumpserver_asset_accounts` | `[]` | Inline accounts: `{name, username, secret_type (password/ssh_key), secret, privileged}`. Can be combined with templates. |

## Example

Full playbook (also in `examples/playbook.yml`).

```yaml
---
# Full example. Usage:
#   ansible-playbook -i examples/inventory.yml examples/playbook.yml
# Remove the hosts again:
#   ansible-playbook -i examples/inventory.yml examples/playbook.yml -e jumpserver_asset_state=absent
- name: Register hosts in JumpServer
  hosts: all
  gather_facts: false
  vars:
    env_file: "{{ lookup('ansible.builtin.file', playbook_dir ~ '/../.env') }}"
  roles:
    - role: "{{ playbook_dir }}/.."
      vars:
        # --- connection ---
        jumpserver_host: "jumpserver-host"
        jumpserver_token: "my-private-token"
        jumpserver_validate_certs: true
        jumpserver_api_timeout: 30

        # --- asset ---
        jumpserver_asset_state: present
        jumpserver_asset_name: "{{ inventory_hostname }}"
        jumpserver_asset_address: "{{ ansible_host | default(inventory_hostname) }}"
        jumpserver_asset_platform: Linux
        jumpserver_asset_protocols:
          - name: ssh
            port: 22
        jumpserver_asset_nodes:
          - /DEFAULT/test-node1/sub-node1
        jumpserver_asset_zone: ""          # e.g. "TestZone" - zone with gateway
        jumpserver_asset_labels: []
        jumpserver_asset_comment: "Managed by Ansible"
        jumpserver_asset_is_active: true

        # --- accounts ---
        jumpserver_asset_account_templates:
          - root
        jumpserver_asset_accounts: []
        # Inline account example (instead of / in addition to a template):
        #  - name: deploy
        #    username: deploy
        #    secret_type: ssh_key
        #    secret: "{{ lookup('ansible.builtin.file', '~/.ssh/id_ed25519') }}"
        #    privileged: false
```

Run it:

```bash
ansible-playbook -i examples/inventory.yml examples/playbook.yml </dev/null
```

`</dev/null` avoids Ansible's "non-blocking stdin" error in some terminals.
Remove a host: add `-e jumpserver_asset_state=absent`.
