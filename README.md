# ovn-heater

Mega script to install/configure/run
a simulated cluster deployed with
[ovn-fake-multinode](https://github.com/ovn-org/ovn-fake-multinode).

**NOTE**: This script is designed to be used on test machines only. It
performs disruptive changes to the machines it is run on (e.g., create
insecure docker registries, cleanup existing docker containers).

# Prerequisites

## Physical topology

* TESTER: One machine to run the tests which needs to be able to SSH
  paswordless (preferably as `root`) to all other machines in the topology.
  Performs the following:
  - prepares the test enviroment: clone the specified versions of `OVS` and
    `OVN` and build the `ovn-fake-multinode` image to be used by the `OVN`
    nodes.
  - provisions all other `OVN` nodes with the required software packages
    and with the correct version of `ovn-fake-multinode` to run simulated/fake
    `OVN` chassis.
  - runs a docker registry where the `ovn-fake-multinode` (i.e.,
    `ovn/ovn-multi-node`) image is pushed and from which all other `OVN`
    nodes will pull the image.
  - runs the scale test scenarios.

* OVN-CENTRAL: One machine to run the `ovn-central` container(s) which
  run `ovn-northd` and the `Northbound` and `Southbound` databases.
* OVN-WORKER-NODE(s): Machines to run `ovn-netlab` container(s), each of
  which will simulate an `OVN` chassis.

The initial provisioning for all the nodes is performed by the `do.sh install`
command. The simulated `OVN` chassis containers and central container are
spawned by the test scripts in `ovn-tester/`.

**NOTE**: `ovn-fake-multinode` assumes that all nodes (OVN-CENTRAL and
OVN-WORKER-NODEs) have an additional Ethernet interface connected to a
single L2 switch. This interface will be used for traffic to/from the
`Northbound` and `Southbound` databases and for tunneled traffic.

**NOTE**: there's no restriction regarding physical machine roles so for
debugging issues the TESTER, OVN-CENTRAL and OVN-WORKER-NODEs can all
be the same physical machine in which case there's no need for the secondary
Ethernet interface to exist.

## Sample physical topology:
* TESTER: `host01.mydomain.com`
* OVN-CENTRAL: `host02.mydomain.com`
* OVN-WORKER-NODEs:
  - `host03.mydomain.com`
  - `host04.mydomain.com`

OVN-CENTRAL and OVN-WORKER-NODEs all have Ethernet interface `eno1`
connected to a physical switch in a separate VLAN, as untagged interfaces.

## Minimal requirements on the TESTER node (tested on Fedora 32)

### Install required packages:
```
dnf install -y git ansible
```

### Make docker work with Fedora 32 (disable cgroup hierarchy):

```
dnf install -y grubby
grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
reboot
```

## Minimal requirements on the OVN-CENTRAL and OVN-WORKER-NODEs

### Make docker work with Fedora 32 (disable cgroup hierarchy):

```
dnf install -y grubby
grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
reboot
```

# Installation

## Get the code:

```
cd
git clone https://github.com/dceara/ovn-heater.git
```

## Write the physical deployment description yaml file:

A sample file written for the deployment described above is available at
`physical-deployments/physical-deployment.yml`.

The file should contain the following mandatory sections and fields:
- `registry-node`: the hostname (or IP) of the node that will store the
  docker private registry. In usual cases this is should be the TESTER
  machine.
- `internal-iface`: the name of the Ethernet interface used by the underlay
  (DB and tunnel traffic). This can be overridden per node if needed.
- `central-node`:
  - `name`: the hostname (or IP) of the node that will run `ovn-central`
    (`ovn-northd` and databases).
- `worker-nodes`:
  - the list of worker node hostnames (or IPs). If needed, worker nodes can
    be further customized using the per-node optional fields described below.

Global optional fields:
- `user`: the username to be used when connecting from the tester node.
  Default: `root`.
- `prefix`: a string (no constraints) that will be used to prefix container
  names for all containers that run `OVN` fake chassis. For example,
  `prefix: ovn-test` will generate container names of the form
  `ovn-test-X-Y` where `X` is the unique part of the worker hostname and `Y`
  is the worker node local index. Default: `ovn-scale`.
- `max-containers`: the maximum number of containers allowed to run on one
  host. Default: 100.

In case some of the physical machines in the setup have different
capabilities (e.g, could host more containers, or use a different ethernet
interface), the following per-node fields can be used to customize the
deployment. Except for `fake-nodes` which is valid only in the context of
worker nodes, all others are valid both for the `central-node` and also for
`worker-nodes`:
- `user`: the username to be used when connecting from the tester node.
- `internal-iface`: the name of the Ethernet interface used for DB and
    tunnel traffic. This overrides the `internal-iface` global configuration.
- `fake-nodes`: the maximum number of containers allowed to run on this
  host. If not specified, the value of `max-containers` from the global
  section is used instead.

## Perform the script installation step:

This generates a `runtime` directory and a `runtime/hosts` ansible inventory
and installs all test components on all required nodes.

```
cd ~/ovn-heater
./do.sh install
```

By default OVS/OVN binaries are built with
`CFLAGS="-g -O2 -fno-omit-frame-pointer"`.

For more aggressive optimizations we can use the `EXTRA_OPTIMIZE` env variable
when installing the runtime. E.g.:

```
cd ~/ovn-heater
EXTRA_OPTIMIZE=yes ./do.sh install
```

## Perform a reinstallation (e.g., new OVS/OVN versions are needed):

```
cd ~/ovn-heater
rm -rf runtime
```

Assuming we need to run a private OVS branch from a fork and a different
OVN branch from another fork we can:

```
cd ~/ovn-heater
OVS_REPO=https://github.com/dceara/ovs OVS_BRANCH=tmp-branch OVN_REPO=https://github.com/dceara/ovn OVN_BRANCH=tmp-branch-2 ./do.sh install
```
## Perform a reinstallation (e.g., install OVS/OVN from rpm packages):

```
cd ~/ovn-heater
rm -rf runtime
```

Run the installation with rpm packages parameters specified:

```
cd ~/ovn-heater
RPM_SELINUX=$rpm_url_openvswitch-selinux-extra-policy RPM_OVS=$rpm_url_openvswitch RPM_OVN_COMMON=$rpm_url_ovn RPM_OVN_HOST=$rpm_url_ovn-host RPM_OVN_CENTRAL=$rpm_url_ovn-central ./do.sh install
```

## Regenerate the ansible inventory:

If the physical topology has changed then update
`physical-deployment/physical-deployment.yml` to reflect the new physical
deployment.

Then generate the new ansible inventory:

```
cd ~/ovn-heater
./do.sh generate
```

# Running tests:

## Scenario definitions

Scenarios are defined in `ovn-tester/ovn_tester.py` and are configurable
through YAML files.  Sample scenario configurations are available in
`test-scenarios/*.yml`.

## Scenario execution

```
cd ~/ovn-heater
./do.sh run <scenario> <results-dir>
```

This executes `<scenario>` on the physical deployment. Current
scenarios also cleanup the environment, i.e., remove all docker containers
from all physical nodes. **NOTE**: If the environment needs to be explictly
cleaned up, we can also execute before running the scenario:

```
cd ~/ovn-heater
./do.sh init
```

The results will be stored in `test_results/<results-dir>`. The results
consist of:
- a `config` file where remote urls and SHA/branch-name of all test components
  (ovn-fake-multinode, ovs, ovn) are stored.
- an `installer-log` where the ouptut of the `./do.sh install` command is
  stored.
- html reports
- a copy of the `hosts` ansible inventory used for the test.
- OVN docker container logs (i.e., ovn-northd, ovn-controller, ovs-vswitchd,
  ovsdb-server logs).
- physical nodes journal files.
- perf sampling results if enabled

## Example (run "scenario #2 - new node provisioning" for 2 nodes):

```
cd ~/ovn-heater
./do.sh run test-scenarios/ovn-low-scale.yml test-low-scale
```

This test will perform two iterations, each of which:
- brings up a fake `OVN` node.
- configures a logical switch for the node and connects it to the
  `cluster-router`.
- configures a logical switch port ("mgmt port") and binds it to an OVS
  internal interface on the fake `OVN` node.
- configures a logical gateway router on the node and sets up NAT to allow
  communication to the outside.
- moves the OVS internal interface to a separate network namespace and
  configures its networking.
- waits until ping from the new network namespace to the "outside" works
  through the local gateway router.

Results will be stored in `~ovn-heater/test_results/test-low-scale/`:
- `config`: remote urls and SHA/branch-names of components used by the test.
- `hosts`: the autogenerated ansible host inventory.
- `logs`: the OVN container logs and journal files from each physical node.
- `*html`: the html reports for each of the scenarios run.

## Example (run "scenario #2 - performing perf analysis for 100 nodes"):

Instruct ovn-heater about the first and last iteration of perf record analysis
providing `ext_cmd_start` and `ext_cmd_stop` in `switch-per-node-100.yml`

```
        farm_nodes: 100
        ports_per_network: 100
        ext_cmd_start: 50
        ext_cmd_stop: 90
        cluster_cmd_path: /root/ovn-heater/runtime/ovn-fake-multinode

```

Results will be stored in `~ovn-heater/test_results/test-100-<date>/logs/<node>-perf/` for each
container.

## Example (run "scenario #3 - scale up number of pods - stress ovn-northd"):

This simulates bringing up 30 `OVN` nodes and binding 1000 pods (logical
ports), distributed across nodes. The test also configures port_groups,
address_sets and ACLs for all pods simulating a network policy that would
allow traffic.

The test is meant to check `ovn-northd` performance so each iteration is
considered successful if `ovn-northd` populated the
Southbound DB, and the logical port is up (`up` is set to `true`).

```
cd ~/ovn-heater
./do.sh run test-scenarios/ovn-30-node-1000-pods.yml test-scale-pods
```

If needed, the number of simulated nodes and total number of simulated pods
can be tweaked, by changing the following fields in
`test-scenarios/ovn-30-node-1000-pods.yml`:

```
farm_nodes: <node-count>
ports_per_network: <total-number-of-pods>
port_wait_type: "none"
port_internal_vm: "False"
```

## Scenario execution with DBs in standalone mode

By default tests configure NB/SB ovsdb-servers to run in clustered mode
(RAFT). If instead tests should be run in standalone mode then the test
scenarios must be adapted by adding the `ovn_cluster_db = False` configuration
and the deployment should be regenerated as follows:

```
cd ~/ovn-heater
CLUSTERED_DB=False ./do.sh generate

echo '        ovn_cluster_db: "False"' >> test-scenarios/ovn-low-scale.yml
./do.sh run <scenario> <results-dir>
```
