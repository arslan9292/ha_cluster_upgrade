# ha_cluster_upgrade

An Ansible role to automate the patching maintenance task for a High Availability Pacemaker Cluster Setup.

This will auto-populate the list of packages that requires for update which are associated with Pacemaker Cluster setup.

Any additional packages (which is not associated with Pacemaker Cluster) can also be included using a pre-defined variable.

## Limitations
 
- Supported OS versions: RHEL 7, RHEL 8 & RHEL 9.
- The `inventory` file _must_ be populated with the cluster node name and _not_ the system hostname.
- The role can be executed on only one cluster setup at once.
- This role performs the patching tasks following the recommended steps as per the 
KCS article from Red Hat:
  - [Recommended Practices for Applying Software Updates to a RHEL High Availability or Resilient Storage Cluster](https://access.redhat.com/articles/2059253)

- The role defaults to `Entire Cluster Update` process detailed under `General Overview of Update Procedures` in the above KBase article as it is more safer approach.

## Requirements

- The cluster status should be in running and in healthy state before executing this Ansible Role.
- The role uses the inventory hostname values while performing `pcs` commands during the playbook execution.

## Role Variables
**_NOTE:_** Any variable starting with two underscores is an internal variable used by the role. The values of these variables should not be modified.

### Defined in `defaults/main.yml`

1.  `rolling_upgrade` (boolean, default: `false`)

It defines the Rolling upgrade method to be used while performing the package upgrade.

2.  `complete_upgrade` (boolean, default: `true`)

It defines the "Entire Cluster Update" method to be used while performing the package upgrade.

**_NOTE:_** If both `rolling_upgrade` and `complete_upgrade` is set to `true`, `rolling_upgrade` takes precedence. If both are undefined in playbook, then role defaults to `Entire Cluster Update` method.

3.  `local_repo` (boolean, default: `true`)

It defines whether to use a local repository for package upgrade.

4.  `release_set` (numeric, default: `null`)

It defines the value to which the system release for subscription-manager is set.

**_NOTE:_** If `local_repo` is set to `false`, then it is optional to set the variable `release_set`. If `release_set` variable is not explicitely configured in `playbook/vars`, then `yum` will update it to the latest release.

5.  `additional_packages_for_update` (list, default: `[]`)

List of additional packages that requires to be updated along with the Pacemaker Cluster packages.

### Defined in `vars/main.yml`

1.  `basic_packages` (list, default: `['pacemaker', 'corosync', 'pcs', 'resource-agents', 'fence-agents-all', 'libqb']`)

List of basic Pacemaker related packages.

2.  `resilient_packages_rh7` (list, default: `['gfs2-utils', 'lvm2-cluster', 'dlm']`)

List of packages associated with Resilient Storage Add-On (GFS2 Cluster Setup) for RHEL 7.

3.  `resilient_packages_rh8_rh9` (list, default: `['gfs2-utils', 'lvm2-lockd', 'dlm']`)

List of packages associated with Resilient Storage Add-On (GFS2 Cluster Setup) for RHEL 8 & 9.

4.  `sap_instance_packages` (list, default: `['resource-agents-sap']`)

List of packages associated with Pacemaker cluter managing SAPInstance.

5.  `sap_hana_packages` (list, default: `['resource-agents-sap-hana']`)

List of packages associated with Pacemaker cluter managing SAPHana.

6.  `qdevice_packages` (list, default: `['corosync-qdevice']`)

List of packages associated with Quorum Device.

7.  `sbd_packages` (list, default: `['sbd']`)

List of packages associated with sbd.

8.  `booth_packages` (list, default: `['booth-site']`)

List of packages associated with Booth cluster setup.

### Details for Rolling Upgrade method

While using rolling upgrade method, the role will detect which is the current active node (or the node which holds the maximum resources) & will start the upgrade from the passive node (i.e. the node which has the least number of resources).

If the cluster setup has a `Master/Slave` resource configured with/without any standalone resource(s), the role will consistently prioritize the status of the `Master/Slave` resource and execute the patch upgrade on the Slave node first.

**For example:**

1. The tasks within the role will calculate the node which holds the least number of resources and start the patch upgrade from that node. Referring to the following resource status, the role will start the patch upgarde from node `rhel-ha-node2.example.com` because it holds least number of resources:
```bash
# pcs resource status
  * dummy1	(ocf::heartbeat:Dummy):	 Started rhel-ha-node2.example.com
  * dummy2	(ocf::heartbeat:Dummy):	 Started rhel-ha-node2.example.com
  * Resource Group: dummy_grp:
    * dummy3	(ocf::heartbeat:Dummy):	 Started rhel-ha-node1.example.com
    * dummy4	(ocf::heartbeat:Dummy):	 Started rhel-ha-node1.example.com
    * dummy5	(ocf::heartbeat:Dummy):	 Started rhel-ha-node1.example.com
```

2. Similarly if cluster is managing a `Master/Slave` resource, the patch upgrade process will consistently commence from the Slave node, regardless of the quantity of resources active on the Master node. In the following state of cluster resources, the patch upgrade will start from node `rhel-ha-node1.example.com` because it is the Slave node:
```bash
# pcs resource status
  * dummy	(ocf::heartbeat:Dummy):	 Started rhel-ha-node2.example.com
  * dummy2	(ocf::heartbeat:Dummy):	 Started rhel-ha-node2.example.com
  * Resource Group: test:
    * dummy3	(ocf::heartbeat:Dummy):	 Started rhel-ha-node1.example.com
    * dummy4	(ocf::heartbeat:Dummy):	 Started rhel-ha-node1.example.com
    * dummy5	(ocf::heartbeat:Dummy):	 Started rhel-ha-node1.example.com
  * Clone Set: state-clone [state] (promotable):
    * Masters: [ rhel-ha-node2.example.com ]
    * Slaves: [ rhel-ha-node1.example.com ]
```

## Example Inventory

The inventory file _must_ be populated with the cluster node name. The inventory host names is used while executing the `pcs` commands.

To get the cluster node names for populating the inventory, use the following command:
```bash
# crm_node -l | awk '{ print $2 }'
```

For example:
```bash
# crm_node -l | awk '{ print $2 }'
rhel-ha-node1.example.com
rhel-ha-node2.example.com

# cat inventory
[cluster]
rhel-ha-node1.example.com
rhel-ha-node2.example.com
```

## Example Playbook

Following examples shows how can different role variables can be used.

### Updating the cluster nodes with additional packages

This example playbook will help to update the cluster nodes with additional packages along with the default cluster packages.
```yaml
- hosts: cluster
  vars:
    additional_packages_for_update:
      - nfs-utils
      - openssh
  roles:
    - ha_cluster_upgrade
```

### Updating the cluster nodes with a specific subscription release

```yaml
- hosts: cluster
  vars:
    local_repo: false
    release_set: 8.6
  roles:
    - ha_cluster_upgrade
```

### Using rolling method for performing the cluster upgrade
```yaml
- hosts: cluster
  vars:
    rolling_upgrade: true
  roles:
    - ha_cluster_upgrade
```

## License
MIT

## Author Information
Arslan Ahmad
