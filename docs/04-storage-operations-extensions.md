# Steps 12–15: Storage and operations extensions

These optional extensions follow the core cloud and disposable VM exercise. Commands labelled **read-only** inspect the cloud. Deployment, failure injection, Terraform apply and cleanup are separate changes that require explicit approval. Never paste credentials, tokens, raw Kolla output, Terraform state or unredacted logs into chat or Git. Cinder is disabled in the current lab, and the exercises do not assume Ceph or usable credentials. If an authentication check cannot run, mark it **BLOCKED**; that is neither a pass nor proof of a broken service.

## Step 12: Plan Cinder block storage

**Goal.** Give VMs persistent block volumes. Cinder provides the disk-management API; its backend is the datastore containing the actual disks. Three local LVM pools have three independent failure domains. They are not replicated or shared storage.

**Status: DESIGN HOLD.** Previous inspection found `cinder-volumes` VGs and thin pools on `/dev/sda` on all three nodes. Recheck device names and actual mappings, and preserve existing data and metadata. The three-node independent-backend design has not been validated against the installed Kolla 2026.2 release's HA precheck. Cinder remains disabled; this guide does not claim installation or backend failover. Do not initialize, wipe, resize or recreate any PV, VG, thin pool or filesystem.

**Where / prerequisites.** Work on the management VM using its installed Kolla virtual environment and `$KOLLA_CONFIG_PATH/multinode`. You need SSH access to each actual node and read-only privilege where required. Use the real existing inventory, not sample hostnames. Review the [Kolla 2026.2 Cinder guidance](https://docs.openstack.org/kolla-ansible/2026.2/reference/storage/cinder-guide.html), including its multi-controller LVM warning.

**Action — read-only discovery on the management VM:**

```bash
source "$KOLLA_VENV/bin/activate"
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph storage
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph cinder-volume
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph cinder-volume-lvm
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph cinder-volume-multiple
```

Inspect the actual group membership and installed logic; they may change across updates. These files contain configuration templates, so do not share output containing secrets.

```bash
sed -n '1,160p' "$KOLLA_VENV/share/kolla-ansible/ansible/roles/cinder/tasks/precheck.yml"
grep -nE 'cinder_cluster_name|cinder_cluster_skip_precheck' "$KOLLA_VENV/share/kolla-ansible/ansible/roles/cinder/defaults/main.yml"
```

A storage group with one host does not by itself prove that the HA check counts only one host. Compare the exact precheck condition with the effective inventory groups. Do not set a common cluster name or skip the check just to make it pass: a common name does not turn local disks into shared storage.

On **each intended storage node**, verify that the prompt names that node. Inspect only, using a local sudo prompt if required:

```bash
hostname
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
findmnt
sudo pvs -o pv_name,pv_size,pv_free,vg_name
sudo vgs -o vg_name,vg_size,vg_free,pv_count
sudo lvs -a -o vg_name,lv_name,lv_size,lv_attr,pool_lv,data_percent,metadata_percent,devices
```

**Expected / stop:** Record each PV-to-VG-to-pool-to-device mapping, identity, mount usage and free capacity without exposing private inventory details publicly. Stop if you find an unexpected device, inaccessible node, missing VG, full pool, inconsistent mapping or unknown existing data. An empty `[storage]` group does not mean the disks are unused. Do not infer device safety from size or an empty mount list.

**Implementation sequence — deferred until separately designed and approved:**

1. Choose explicitly between three independent local backends, one local backend, or a properly supported shared/replicated backend. If a host with an independent local backend is lost, its volumes become unavailable; a single local backend is not HA either. Document failure domains, placement/scheduling, capacity, iSCSI paths, backups, recovery and rollback. Verify driver and Cinder service compatibility with the release. A single-host alternative still needs inventory and precheck validation.
2. In an isolated test environment or reviewed configuration, determine exact service placement and the groups counted by the HA precheck for the chosen design. Preserve the installed inventory's child relationships. Establish an independently reviewed, reproducible configuration. **This guide does not authorize a bypass or shared cluster name.** Stop if the configuration cannot be demonstrated.
3. Verify backup coverage and a restore test for all potentially affected data. Also verify target identities, capacity, rollback, privilege and service impact. Review `enable_cinder_backup` and `cinder_backup_driver`: the installed lab defaults enable backup and select Ceph when backup is enabled, but this lab has no Ceph. Explicitly disable backup or configure a supported, separately provisioned backup target. Snapshots are not off-host backups.
4. Only after separate change approval, back up configuration securely and change the approved inventory and globals. Validate the configuration, run release-matched Kolla prechecks, then deploy using the existing privileged execution procedure. Stop on any failure. Prechecks alone do not validate data access. This guide supplies no commands to alter existing LVM while the design is unresolved.
5. With authorized disposable resources, check `openstack volume service list` and `openstack volume pool list --detail`. Verify that the expected services are up and each intended pool is present. On **each** intended backend, create a small test volume using a verified backend-specific volume type. Wait for `available`, attach it to the approved test VM, and confirm the guest detects the correct disk without formatting an existing disk. Detach it, verify `available`, and delete only the test volume. Record the backend, volume ID, VM ID and results. Registration alone is not a successful I/O test. Test backup/restore separately if configured.

**Completion gate:** All approved backends must pass create, attach, guest visibility and detach tests, as well as documented recovery tests. Otherwise, mark the untested portions DEFERRED. Continue to Step 13 without Cinder if necessary.

## Step 13: HA failure testing and recovery

**Goal.** Measure how a specific redundant component behaves under a controlled failure. Do not assume that three nodes or a VIP make every service HA. **Status: no failover or recovery test is established by this document.** Storage failover has not been demonstrated.

**Where / prerequisites.** Use the management VM and a user-visible terminal. You need console/out-of-band access to all nodes, verified current backup/restore coverage, a disposable workload, a maintenance window and a named operator. The exact target, action and rollback also require separate approval. Confirm that no important workload depends on the test. Whole-node shutdown is not a default test, and do not subject Galera or RabbitMQ to generic service restarts.

**Action — read-only baseline:** Run these commands on the management VM with a configured authenticated cloud name, without printing the token. If authentication is unavailable, stop and mark API checks BLOCKED:

```bash
openstack --os-cloud "$OS_CLOUD" server list --all-projects
openstack --os-cloud "$OS_CLOUD" compute service list
openstack --os-cloud "$OS_CLOUD" network agent list
```

The cloud name is an example. Check the names in your protected local `clouds.yaml` privately and use the actual name. Record the expected node identities, each service's `up`/`enabled` state and the disposable VM's status. Inspect the [existing shutdown and recovery guide](https://github.com/DinosaurKeys/openstack-lab/blob/main/docs/SHUTDOWN-AND-RECOVERY.md) and its scripts before relying on them. Independently verify every configured Galera member's current Primary/Synced/ready status and RabbitMQ membership and partitions. A running container, reachable VIP or single successful API request does not establish quorum. Do not print credentials or raw logs.

**Proposed controlled test — only after approval:** Select one non-database, non-broker service with independently confirmed redundant peers. Verify which backend the request reaches and capture baseline direct-backend and VIP responses. Write down the exact failure injection, maximum outage, stop conditions, restoration command and owner. Inject only that one failure and observe client errors, timing and surviving backend identity. Restore exactly what was changed, then repeat the baseline and exercise the disposable workload. Stop immediately if quorum, other peers or recovery is uncertain; do not proceed to another fault. A test must not imply volume failover or full power-cycle recovery.

**Result / stop:** A PASS requires recorded before, during and after observations, a healthy restored component and an independently successful workload check. Record the duration and scope. Mark FAIL if any assertion fails, BLOCKED if credentials, member status or console/backup prerequisites are missing, and NOT TESTED if no fault was injected. Do not run database recovery unless a separately diagnosed complete outage and exact recovery action are approved. Follow the linked procedure only after verifying that it matches the current deployment.

## Step 14: Terraform sandbox

**Goal.** Describe a disposable cloud resource in code, review a plan, then compare it with the actual state. Terraform state is sensitive and is not a backup of the cloud.

**Where / prerequisites.** Use the management VM after a manual VM/network test has passed. You need a separately authorized sandbox project, a restricted application credential and quotas. Use [HashiCorp's install instructions](https://developer.hashicorp.com/terraform/install) and the [OpenStack provider documentation](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs). Installing software, creating credentials and creating any cloud resource are changes, not read-only checks.

**Action — read-only checks:**

```bash
terraform version
openstack --os-cloud "$OS_CLOUD" token issue -c expires -f value
```

Use the real configured cloud name rather than the example. The second command requests a token but displays only its expiry, not the token itself. If either command or authentication fails, mark the step BLOCKED. Do not print credentials to troubleshoot.

**Implementation — use only a separately approved sandbox project:**

1. Use the cloud's approved procedure to create a protected sandbox application credential. Put it in a private `clouds.yaml` entry named `lab-sandbox`; never commit or display it. Choose and protect the state location. This exercise uses a local state file in a private directory, not a public Git repository. Do not point it at an existing shared network.
2. On the management VM, create a new working directory. If the command says the directory exists, stop and inspect it rather than using or overwriting it:

   ```bash
   mkdir -m 700 "$HOME/terraform-openstack-sandbox"
   ```

   Only after that succeeds, open the configuration file:

   ```bash
   cd "$HOME/terraform-openstack-sandbox"
   nano main.tf
   ```

3. Put the following **configuration, not terminal commands**, into `main.tf`. Save with Ctrl+O, Enter and Ctrl+X. The configuration plans one new, uniquely named network with no subnet or router. The provider version is pinned for reproducibility. Review and deliberately update it later rather than silently taking a new release:

   ```hcl
   terraform {
     required_providers {
       openstack = {
         source  = "terraform-provider-openstack/openstack"
         version = "= 3.4.0"
       }
     }
   }

   provider "openstack" {
     cloud = "lab-sandbox"
   }

   resource "openstack_networking_network_v2" "practice" {
     name           = "terraform-practice-network"
     admin_state_up = true
   }
   ```

4. Initialize the pinned provider. This downloads software and writes files in the sandbox directory. Stop on errors:

   ```bash
   terraform init
   ```

   Check formatting and validate the configuration:

   ```bash
   terraform fmt -check
   terraform validate
   ```

   Stop if either check fails. Validation does not prove cloud connectivity. Next, review the plan in the private terminal:

   ```bash
   terraform plan
   ```

   Confirm the sandbox project and check that the plan creates exactly the intended network, with no unexpected updates or deletions. Plans and state can contain sensitive values; do not paste them into chat.
5. Only with explicit approval, run apply and review Terraform's fresh confirmation prompt:

   ```bash
   terraform apply
   ```

   After apply succeeds, list networks in the **sandbox project**, identify the exact network ID and inspect that ID. Do not assume an exit code proves the network is usable:

   ```bash
   openstack --os-cloud lab-sandbox network list
   ```

   ```bash
   openstack --os-cloud lab-sandbox network show '<verified-network-id>'
   ```

   Replace the quoted placeholder with the actual ID; do not type the angle brackets.
6. To remove the network, obtain explicit destroy approval and inspect `terraform plan -destroy`. Confirm it includes only resources owned by this sandbox state. Run `terraform destroy` only after review, then verify that the exact network ID is absent. Never destroy shared networks or resources created outside this exercise.

Do not commit `.terraform/`, state files, plans, `clouds.yaml` or credentials. A provider lock file is distinct from these secrets, but review it before committing. You can review this recipe's syntax without creating resources. Do not claim a live apply or destroy unless it actually succeeded.

## Step 15: Prometheus and Grafana

**Goal.** Use Prometheus to collect metrics and Grafana to display them. A dashboard is useful only when its data source and scrape targets work. **Status: deployment and monitoring validation deferred.**

**Where / prerequisites.** Work on the management VM and intended monitoring hosts. You need a stable core cloud, explicitly chosen real inventory hosts, adequate RAM and disk, a retention and access plan, secure credentials and a separately approved change. Do not assume the sample monitoring host exists. Keep endpoints on the management network or use an approved authenticated encrypted access path.

**Action — read-only discovery on the management VM:**

```bash
source "$KOLLA_VENV/bin/activate"
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph monitoring
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph prometheus
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph grafana
```

Review the installed release's `roles/prometheus/defaults/main.yml`, prechecks, relevant service inventory children and nonsecret monitoring flags in globals. Do not dump passwords. On each intended host, inspect `free -h`, `df -h` and current load (`uptime`). Compare available capacity and retention against the proposed workload rather than assuming idle services free memory. This guide does not select nodes or make a capacity decision.

**Implementation sequence — after approval:** Choose hosts, failure and retention expectations, access restrictions, exporter scope and rollback. Back up the configuration securely. Add real host aliases to the correct release-matched inventory groups without damaging child relationships. Review and set `enable_prometheus: true` and `enable_grafana: true` in the existing globals rather than creating duplicate keys. Validate the YAML and effective placement. Use the matching Kolla environment and approved privilege mode. Run release-matched `kolla-ansible prechecks -i $KOLLA_CONFIG_PATH/multinode`, adding `--use-test-images` only for the approved test-image configuration. Stop on failure. After reviewing impact, run the approved Kolla deploy, not an unreviewed host-limited operation. Kolla output can contain secrets.

**Validation gate:** Through the approved authenticated path, confirm Grafana login, Prometheus data source connectivity, scrape-target status for each expected exporter and fresh timestamped host metrics in a dashboard. Confirm that access is denied on unintended networks. Test alerts only if alerting is configured. A running container or login page alone is not a metrics test. If authentication, a target or fresh data is missing, report the exact portion BLOCKED or FAIL and stop before relying on alerts.

**References:** [Kolla 2026.2 operating guidance](https://docs.openstack.org/kolla-ansible/2026.2/user/operating-kolla.html), [Prometheus querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/), and [Grafana data sources](https://grafana.com/docs/grafana/latest/datasources/). Review installed role defaults in addition to upstream documentation; availability and defaults can differ by release.
