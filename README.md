# OpenStack Lab Guide

**Draft for Joe's review, not yet an approved installation procedure.**

This guide walks through building and operating a small, three-node OpenStack lab with Ubuntu Server 24.04 and Kolla-Ansible. Work through the checkpoints as you go. An `ACTIVE` service or VM is encouraging, but it does not prove that every network, storage or recovery path works.

## Start in the right place

| Your situation | Start here |
|---|---|
| Fresh build using R2D2 | [Step 00: Setup Management VM](docs/00-management-vm.md) |
| Existing cloud | [Existing-cloud checkpoint](#existing-cloud-checkpoint) |
| Something failed | Stop at the failing step. Preserve the error without sharing secrets, investigate it, and recover the current cloud before considering a rebuild. |

> A block marked `bash` contains shell commands. Run them only in the stated location and after the preceding checks pass. Pictures, `yaml`/`ini`/`hcl` configuration, diagrams, tables and example output are for reading or editing the specified file, not for typing into a terminal.

## How to follow a resource-creation step

When a step creates a cloud object, do not paste the entire section into the terminal. The guide uses this sequence deliberately:

```text
inspect existing objects
        ↓
create one object, if absent
        ↓
record the returned ID
        ↓
verify that ID and its dependencies
        ↓
continue to the next object
```

If a command fails, stop. Do not rerun a create command until you have checked whether the first attempt created the object anyway. A placeholder such as `<returned-network-id>` is documentation input, not a value to type literally.

## The lab at a glance

![Three-node lab topology](diagrams/lab-topology.svg)

The management VM runs the deployment tools, and each of the three physical nodes has control, network and compute roles. Each node's management NIC carries management and API traffic; its USB NIC connects the provider network to the home LAN. Separate SATA SSDs are intended for local storage, but local LVM is **not replicated HA storage**.

### Network plan

| Item | Planned value | What it is |
|---|---|---|
| Home LAN | `192.168.0.0/24` | The physical network. |
| Home router | `192.168.0.1` | The LAN's upstream gateway, confirmed on the node management routes. |
| Node management addresses | `192.168.0.200–202` | One address per physical node. |
| OpenStack API VIP | `192.168.0.150` | Shared address for the OpenStack APIs, not a VM floating IP. |
| Proposed floating-IP pool | `192.168.0.151–199` | Joe reports this range is outside DHCP and available. Verify current allocations before assigning any address. Router gateway ports also consume an address from the pool. |
| Private VM subnet | `10.254.211.0/24` | Private OpenStack network. Confirm it does not overlap connected VPNs or other networks before creating it. |
| Provider mapping | Planned `physnet1` → `br-ex` → `provider0` | Verify the actual USB NIC, bridge and physical-network mapping during Step 4; the current node discovery showed the USB NICs down, so this is not yet verified. |
| Timezone | `Europe/Bucharest` | Lab display timezone; time synchronization remains important on all hosts. |

These values are drafts for this lab, not universal defaults. Being outside the DHCP range does not prove that every address in the pool is unallocated. For the actual installation, also confirm the management VM's address, provider switch/VLAN, client source address and upstream reachability. **Never put passwords, private keys or cloud credentials in this repository.**

## One learning path

| Step | Outcome |
|---|---|
| [00. Setup Management VM](docs/00-management-vm.md) | Prepare a separate workspace on R2D2 without overwriting the old configuration. |
| [1. Ubuntu and recovery access](docs/01-foundations.md#step-1-install-and-identify-the-systems) | Identify the correct OS disk, install Ubuntu and verify console recovery. |
| [2. SSH and privilege access](docs/01-foundations.md#step-2-establish-ssh-access) | Establish key-based management access and verified non-interactive sudo on each system. |
| [3. Management tools](docs/01-foundations.md#step-3-install-kolla-ansible-on-the-manager) | Install Ansible and Kolla-Ansible on the management VM. |
| [4. Network and disk plan](docs/01-foundations.md#step-4-plan-and-configure-node-networking-with-ansible) | Record interfaces, addresses, VIP, floating-IP pool and storage identities. |
| [5. Kolla inventory](docs/02-deploy-openstack.md#step-5-build-and-check-the-multinode-inventory) | Target the three actual nodes in the correct groups. |
| [6. Hostnames and time](docs/02-deploy-openstack.md#step-6-verify-names-and-time) | Verify resolution, timezone and clock synchronization. |
| [7. Kolla configuration](docs/02-deploy-openstack.md#step-7-configure-globals-and-protect-passwords) | Configure interfaces and VIP; protect secrets. |
| [8. Bootstrap and prechecks](docs/02-deploy-openstack.md#step-8-verify-sudo-bootstrap-and-precheck) | Prepare hosts and stop on failing prerequisites. |
| [9. Deploy and verify](docs/02-deploy-openstack.md#step-9-deploy-and-validate) | Deploy the control plane and verify the reported APIs and agents. |
| [Recovery after shutdown/reboot](docs/SHUTDOWN-AND-RECOVERY.md) | Perform read-only post-reboot checks, make an evidence-based recovery decision, and restart one approved service when required. |
| [10. First VM and client](docs/03-first-vm-and-network.md#step-10-create-the-first-test-vm) | Configure the OpenStack CLI client, then create and verify a private network, image, keypair and small VM one resource at a time. |
| [11. Provider network and floating IP](docs/03-first-vm-and-network.md#step-11-connect-the-private-network-to-the-provider-lan) | Verify or create the external network, subnet and router in dependency order; then allocate and associate one floating IP and test each layer. |
| [12. Cinder storage](docs/04-storage-operations-extensions.md#step-12-plan-cinder-block-storage) | Inspect the existing SSDs and validate the LVM design before enabling storage. |
| [13. Failure and HA testing](docs/04-storage-operations-extensions.md#step-13-ha-failure-testing-and-recovery) | Plan and explicitly approve a controlled failure exercise. |
| [14. Terraform](docs/04-storage-operations-extensions.md#step-14-terraform-sandbox) | Learn infrastructure as code after the manual cloud tests. |
| [15. Monitoring](docs/04-storage-operations-extensions.md#step-15-prometheus-and-grafana) | Plan Prometheus and Grafana and verify actual metrics. |

Steps 12–15 are follow-on exercises. Leave a storage design disabled until it has been validated; do not force it past a precheck.

## Existing-cloud checkpoint

**Do not run the fresh-install steps against an existing cloud.** In the previous deployment, a read-only check found `lab-smoke` `ACTIVE` with private address `10.254.211.204`, but no external Neutron network, router or floating IP. The VM used the `default` security group, and no SSH key was injected when it was created. Joe has since reinstalled the three nodes. These observations are historical and do not establish the current state.

For Joe's fresh build, start with [Step 00](docs/00-management-vm.md), then identify the reinstalled nodes in Step 1. Do not assume the old VM or networks still exist. The [VM and networking chapter](docs/03-first-vm-and-network.md) explains the fresh-build path and how an existing VM differs.

For a deployed cloud, use the [Shutdown, reboot and recovery guide](docs/SHUTDOWN-AND-RECOVERY.md) after checking that its inventory, Kolla release, container engine and service placement match the current deployment. Database recovery is not a routine action after every reboot.

## Safety and verification

- Before changing a disk, network or service, review the exact target, impact, backup and rollback. Get explicit approval for real changes.
- Do not run `pvcreate` or `vgcreate`, reinstall Ubuntu, delete a VM or restart a service to fix an unexplained failure.
- Run commands one step at a time. Stop on errors, and do not paste example output into the shell.
- Record whether each check **passed**, **failed**, is **blocked** or was **not tested**. Syntax checks and simulated tests are not a live deployment.
- This guide is a draft for review. Three nodes and a VIP do not by themselves make every OpenStack component or its local storage highly available.
