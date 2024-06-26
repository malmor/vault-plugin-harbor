# Vault Plugin: Harbor robot account
[![GitHub license](https://img.shields.io/github/license/manhtukhang/vault-plugin-harbor.svg)](https://github.com/manhtukhang/vault-plugin-harbor/blob/main/LICENSE)
[![Release](https://img.shields.io/github/release/manhtukhang/vault-plugin-harbor.svg)](https://github.com/manhtukhang/vault-plugin-harbor/releases/latest)
[![Lint](https://github.com/manhtukhang/vault-plugin-harbor/actions/workflows/golangci-lint.yml/badge.svg)](https://github.com/manhtukhang/vault-plugin-harbor/actions/workflows/golangci-lint.yml)
[![Integration Test](https://github.com/manhtukhang/vault-plugin-harbor/actions/workflows/integration-test.yml/badge.svg)](https://github.com/manhtukhang/vault-plugin-harbor/actions/workflows/integration-test.yml)
[![Security scanning](https://github.com/manhtukhang/vault-plugin-harbor/actions/workflows/snyk.yml/badge.svg)](https://github.com/manhtukhang/vault-plugin-harbor/actions/workflows/snyk.yml)
[![Go Report Card](https://goreportcard.com/badge/github.com/manhtukhang/vault-plugin-harbor)](https://goreportcard.com/report/github.com/manhtukhang/vault-plugin-harbor)
[![Maintainability](https://api.codeclimate.com/v1/badges/08cc4f11bf1bbb09b7a0/maintainability)](https://codeclimate.com/github/manhtukhang/vault-plugin-harbor/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/08cc4f11bf1bbb09b7a0/test_coverage)](https://codeclimate.com/github/manhtukhang/vault-plugin-harbor/test_coverage)

[Vault](https://www.vaultproject.io) plugin for [(Go)Harbor](https://goharbor.io) robot account dynamic generating

This is a [Vault plugin](https://www.vaultproject.io/docs/internals/plugins.html)
and is meant to work with Vault. This guide assumes you have already installed Vault
and have a basic understanding of how Vault works. Otherwise, first read this guide on
how to [get started with Vault](https://www.vaultproject.io/intro/getting-started/install.html).

## Install plugin
- Download plugin from [release page](https://github.com/manhtukhang/vault-plugin-harbor/releases)

- Unarchive and copy to the plugins dir on all Vault servers
  ```bash
  $ tar xzf vault-plugin-harbor_<version>_<os>_<arch>.tar.gz
  $ rsync/cp vault-plugin-harbor <vault-installed-path>/plugins
  ```

- Get plugin's SHA256 checksum
  ```bash
  SHA256=$(sha256sum vault-plugin-harbor | cut -d ' ' -f1)
  ```

- Register plugin to Vault secret engine
  ```bash
  $ vault plugin register \
        -sha256=$SHA256 \
        -command=vault-plugin-harbor \
        secret harbor
  # Example:
  $ vault plugin register \
        -sha256=$SHA256 \
        -command=vault-plugin-harbor \
        secret harbor
  ```

> [!NOTE]
> [Please follow the official docs here!](https://developer.hashicorp.com/vault/docs/plugins/plugin-architecture#plugin-registration)

## Upgrade plugin version
- Download and install/register a new version of this plugin with the above installation steps

- Tune the existing mount to configure it to use the newly registered version
  ```bash
  $ vault secrets tune -plugin-version=v<new-version> <mount-path>
  # Example:
  $ vault secrets tune -plugin-version=v1.0.1 harbor/
  ```

- Reload plugin
  ```bash
  $ vault plugin reload -plugin harbor
  ```

> [!NOTE]
> [Please follow the official docs here!](https://developer.hashicorp.com/vault/docs/upgrading/plugins#upgrading-vault-plugins)

## Usage
- Mount harbor plugin
  ```bash
  $ vault secrets enable -path <mount-path> harbor
  # Example:
  $ vault secrets enable -path harbor/ harbor
  ```

- Write harbor config
  ```bash
  $ vault write \
        <mount-path>/config url=<harbor-url> \
        username=<harbor-admin-username> \
        password=<harbor-admin-password>
  # Example:
  $ vault write \
        harbor/config url="https://harbor.internal.domain" \
        username="admin" \
        password="aStronggPw123"
  ```

- Create a role for robot account

  + Create a json file for role permissions definition [Details](#role-definition)

    Example: `role-permissions.json`
    ```json
    [
      {
        "namespace": "project-a",
        "kind": "project",
        "access": [
          {
            "action": "pull",
            "resource": "repository"
          },
          {
            "action": "push",
            "resource": "repository"
          },
          {
            "action": "create",
            "resource": "tag"
          },
          {
            "action": "delete",
            "resource": "tag"
          }
        ]
      },
      {
        "namespace": "project-b",
        "kind": "project",
        "access": [
          {
            "action": "pull",
            "resource": "repository"
          }
        ]
      }
    ]
    ```

  + Write role (create if not existed/ upgrade if existed)
    ```bash
    $ vault write \
            <mount-path>/roles/<role-name> \
            ttl=<time-to-live> \
            max_ttl=<max-time-to-live> \
            permissions=@<role-permissions-json-file>
    # Example:
    $ vault write \
            harbor/roles/test-role \
            ttl=60s \
            max_ttl=10m \
            permissions=@role-permissions.json
    ```

- Get robot account (and its secret/credential) from the created role
  ```bash
  $ vault read <mount-path>/creds/<role-name>
  # Example:
  $ vault read harbor/creds/test-role

  Key                         Value
  ---                         -----
  lease_id                    harbor/creds/test-roles/Wxidlpz1tVrb18XL7Zg4vPZM
  lease_duration              1m
  lease_renewable             true
  robot_account_auth_token    cm9ib3QkdmF1bHQudGVzdC1yb2xlcy5yb290LjE2NTc5NjQ0NjkwNjkyODkzOTE6RE93bXNnN2pEVEZmVlJoWWFwM3BMY0FJdjJIYkJycFg=
  robot_account_id            415963
  robot_account_name          robot$vault.test-roles.root.1657964469069289391
  robot_account_secret        DOwmsg7jDTFfVRhYap3pLcAIv2HbBrpX
  ```
  [Credential output struct explaining](#robot-account-credential-output-struct)

### Role definition
- Each role contains a list of Harbor robot account's permissions
- Robot permission struct ([source](https://github.com/goharbor/go-client/blob/main/pkg/sdk/v2.0/models/robot_permission.go#L20-L30))
  ```json
  {
      "namespace": "<namespace>",
      "kind": "<kind>",
      "access": "[<access>]"
  }
  ```
  | Attribute | Type | Value | Description |
  |:----------|:-----|:------|:------------|
  | `kind` | string | `system`\|`project` | scope of permission |
  | `namespace` | string | `/`\|`*`\|`<project-name>` | when `kind=system`, this field must be `/` only; when `kind=project`, `*` means all projects |
  | `access` | list of access struct | | access list |

- `access` struct ([source](https://github.com/goharbor/go-client/blob/main/pkg/sdk/v2.0/models/access.go#L18-L28))
  ```json
  {
      "action": "<action>",
      "resource": "<resource>",
      "effect": "<effect>"
  }
  ```
  | Attribute | Type | Value | Description |
  |:----------|:-----|:------|:------------|
  | `action` | string | [possible values](https://github.com/goharbor/harbor/blob/main/src/common/rbac/const.go#L20-L36) | action name, `*` means all actions |
  | `resource` | string | [possible values](https://github.com/goharbor/harbor/blob/main/src/common/rbac/const.go#L39-L81) | resource name, `*` means all resources |
  | `effect` | string | `allow`\|`deny` | effect of the access (allow or deny) |

>[!NOTE]
>The `resource` and `action` mapping is depended on what kind of permission (`system` or `project`),
>view more detailed mappings at: [system](https://github.com/goharbor/harbor/blob/main/src/common/rbac/const.go#L85-L155), [project](https://github.com/goharbor/harbor/blob/main/src/common/rbac/const.go#L156-L229)


### Robot account credential output struct
| Key Name | Description |
|:----|:------------|
| `lease_id` | Vault [lease](https://www.vaultproject.io/docs/concepts/lease) ID (with full path) |
| `lease_duration` | Vault [lease duration](https://www.vaultproject.io/docs/concepts/lease#lease-durations-and-renewal) |
| `lease_renewable` | As its name |
| `robot_account_id` | Robot account ID generated from Harbor API |
| `robot_account_name` | Robot account name generated from Harbor API |
| `robot_account_secret` | Robot account secret (password) generated from Harbor API |
| `robot_account_auth_token` | Robot account base64 token, combined from above `robot_account_name` and `robot_account_secret` (base64(robot_account_name:robot_account_secret))|


# Is this useful to you?
<a href='https://ko-fi.com/manhtukhang' target='_blank'><img height='48' src='https://cdn.ko-fi.com/cdn/kofi3.png?v=3' alt='Buy Me a Coffee at ko-fi.com' style='border:0px;height:48px;' /></a>
<a href='https://www.buymeacoffee.com/manhtukhang' target='_blank'><img height='48' src='https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png' alt='Buy Me a Coffee at buymeacoffee.com' style='height: 48px;' ></a>
