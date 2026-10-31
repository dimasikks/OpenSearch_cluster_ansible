# Ansible OpenSearch Cluster

Ansible role for deploying and managing an OpenSearch cluster lifecycle: initial installation, node addition and removal, and full cluster teardown for Centos 9.

## Features

- Install and initialize an OpenSearch cluster with health checks, plugin verification and authentication checks
- Automatically create internal users defined in `install.yml` and `templates/internal_users.j2`, with password generation and hashing, deployed to the first master node (`first_master: true`)
- Generate per-node TLS certificates signed by a self-signed CA
- Install and configure OpenSearch Dashboards on the first master node (`first_master: true`)
- Install the security plugin
- Remove the cluster
- Add a node to the cluster
- Remove a node from the cluster

## Configuration

For a successful install, uninstall, or node add/remove operation, you must correctly fill in `host_vars` for every host in the `opensearch` group, configure the role defaults, and define the inventory with the group and required variables. Make sure nodes can reach each other, and that a dedicated block device for logging and data storage is available if you use the `mount` variable (see below).

### Inventory example (`hosts`)

```yaml
all:
  children:
    servers:
      hosts:
        test1: {}
        test2: {}
        test3: {}
    opensearch:                      # OpenSearch cluster group
      hosts:
        test1: {}                    # all hosts must be listed during initial setup
        test2: {}
        test3: {}
```

### Defaults example

```yaml
cluster:
  name: CLUSTER-NAME                 # human-readable cluster name

mount: /path/to/disk                 # block device path for OpenSearch and OpenSearch Dashboards data
```

### `host_vars` example (for **every host**)

```yaml
first_master: true                   # set on exactly one VM, see below

node:
  roles: [ "cluster_manager" ]       # roles for this specific node

vm:
  max_map_count: 262144              # memory optimization parameters
```

### The `first_master` parameter

Set on exactly one of the master nodes. It marks the node where OpenSearch Dashboards is installed. On this node, `/etc/opensearch/ssl` contains the target self-signed CA and the certificates for every node, along with the scripts used to generate them. The same node also stores the `passwords` file for the created users under `/etc/opensearch`.

> **Important:** When removing this node, note that the root CA and passwords are not distributed to other nodes. It is strongly recommended to back them up to a secure store (e.g. Vault) right after initialization. If you must remove this node:
>
> 1. Back up the CA and passwords.
> 2. Set `first_master: true` on the replacement node.

## Adding and removing nodes

To add or remove a node, set the corresponding variable in the inventory for that specific host. Only one node can be added or removed per run.

### Add node example

```yaml
opensearch:                      # OpenSearch cluster group
      hosts:
        test1: {}                # all hosts must be listed during initial setup
        test2: {}
        test3: {}
        test4:                   # new host to add
          add_node: true         # marks the host to be added
```

Result: node `test4` is added.

### Remove node example

```yaml
opensearch:                      # OpenSearch cluster group
      hosts:
        test1: {}                # all hosts must be listed during initial setup
        test2:                   # host to remove
          del_node: true         # marks the host to be removed
        test3: {}
        test4:                   # previously added host; ignored during a removal run,
          add_node: true         # may be left as-is
```

Result: node `test2` is removed.

---

Before every add/remove run, make sure that **exactly one** host has the corresponding variable set. For an add operation, only the new node should have `add_node`. For a remove operation, only the target node should have `del_node`.

## Usage

### install

Initial cluster setup: certificate generation, cluster configuration, health checks, OpenSearch Dashboards installation, and user creation.

```bash
ansible-playbook -i inventory playbooks/opensearch.yml --tags install
```

### uninstall

Removes the cluster and everything related to OpenSearch and OpenSearch Dashboards on all nodes.

```bash
ansible-playbook -i inventory playbooks/opensearch.yml --tags uninstall
```

### add_node

Adds a node to the cluster and refreshes the configuration of all VMs in the `opensearch` inventory group.

```bash
ansible-playbook -i inventory playbooks/opensearch.yml --tags add_node
```

### del_node

Removes a node from the cluster and refreshes the configuration of all VMs in the `opensearch` inventory group.

```bash
ansible-playbook -i inventory playbooks/opensearch.yml --tags del_node
```

## Additional notes

- HAProxy is installed additionally to proxy traffic to OpenSearch Dashboards.
- The `install` tag is idempotent and refreshes the configuration on every run.
- Certificates and user passwords are generated **only once** during the initial install and are stored on the node marked `first_master: true`.
- Removing a node only removes it from the cluster and the corresponding configuration; the removed node is **not** cleaned of OpenSearch components.
- After every action except `uninstall`, a debug output with cluster health checks is printed. If any health check fails, the configuration is incorrect — inspect the cluster logs for details.
- As mentioned above, after the initial setup you should store the root CA (`/etc/opensearch/ssl/root-ca*`) and the passwords file (`/etc/opensearch/passwords`) in a secure location. If this node is removed and no backups exist, access to the cluster will be lost.