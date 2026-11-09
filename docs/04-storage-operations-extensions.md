# Steps 12–15 — Storage and operations extensions

These are optional extensions after the core cloud and disposable VM exercise work. The commands labelled **read-only** inspect the cloud; proposed deployment, failure injection, Terraform apply and cleanup are separate changes and require explicit approval. Never paste credentials, tokens, raw Kolla output, Terraform state or unredacted logs into chat or Git. The present lab has Cinder disabled; no Ceph or usable credentials are assumed. A blocked authentication check is **BLOCKED**, not a pass or proof of a broken service.

## Step 12 — Plan Cinder block storage

**Goal.** Give VMs persistent block volumes. Think of Cinder as the disk-management API, and its backend as the datastore containing the actual disks. Three local LVM pools are three independent failure domains, not replicated or shared storage.

**Status: DESIGN HOLD.** Previous inspection identified existing `cinder-volumes` VGs and thin pools on `/dev/sda` on all three nodes. Device names and actual mappings must be rechecked; existing data and metadata must be preserved. The three-node independent-backend design has not been validated against this installed Kolla 2026.2 release's HA precheck. Cinder remains disabled; no installation or backend failover is claimed. Do not initialize, wipe, resize or recreate any PV, VG, thin pool or filesystem.

**Where / prerequisites.** Management VM, using its installed Kolla virtual environment and `/etc/kolla/multinode`; SSH access to each actual node and read-only privilege where needed. Work from the real existing inventory, not sample hostnames. Review [Kolla 2026.2 Cinder guidance](https://docs.openstack.org/kolla-ansible/2026.2/reference/storage/cinder-guide.html), including its multi-controller LVM warning.

**Action — read-only discovery on the management VM:**

```bash
source "$HOME/.venvs/kolla-2026.2/bin/activate"
ansible-inventory -i /etc/kolla/multinode --graph storage
ansible-inventory -i /etc/kolla/multinode --graph cinder-volume
ansible-inventory -i /etc/kolla/multinode --graph cinder-volume-lvm
ansible-inventory -i /etc/kolla/multinode --graph cinder-volume-multiple
```

Inspect actual group membership and the installed logic, which may change across updates. These files contain configuration templates; do not share output containing secrets.

```bash
sed -n '1,160p' "$HOME/.venvs/kolla-2026.2/share/kolla-ansible/ansible/roles/cinder/tasks/precheck.yml"
grep -nE 'cinder_cluster_name|cinder_cluster_skip_precheck' "$HOME/.venvs/kolla-2026.2/share/kolla-ansible/ansible/roles/cinder/defaults/main.yml"
```

A storage group with one host alone does not prove the HA check counts only one host. Compare the exact precheck condition and effective inventory groups; do not set a common cluster name or skip the check just to make it pass. A common name does not turn local disks into shared storage.

On **each intended storage node**, verify the prompt names that node; inspect only, using a local sudo prompt if required:

```bash
hostname
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
findmnt
sudo pvs -o pv_name,pv_size,pv_free,vg_name
sudo vgs -o vg_name,vg_size,vg_free,pv_count
sudo lvs -a -o vg_name,lv_name,lv_size,lv_attr,pool_lv,data_percent,metadata_percent,devices
```

**Expected / stop:** record each PV-to-VG-to-pool-to-device mapping, identity, mount usage and free capacity without exposing private inventory details publicly. Stop on an unexpected device, inaccessible node, missing VG, full pool, inconsistent mapping or unknown existing data. Empty `[storage]` does not mean the disks are unused. Do not infer device safety from size or an empty mount list.

**Implementation sequence — deferred until separately designed and approved:**

1. Decide explicitly between three independent local backends (a host loss makes its own volumes unavailable), one local backend (also not HA), or a properly supported shared/replicated backend. Document failure domains, placement/scheduling, capacity, iSCSI paths, backups, recovery and rollback. Verify driver and Cinder service compatibility with the release; a single-host alternative still needs inventory/precheck validation.
2. In an isolated test environment or reviewed configuration, resolve exact service placement and the HA precheck's counted groups for the chosen design. Preserve the installed inventory's child relationships. Establish an independently reviewed, reproducible configuration; **no bypass or shared cluster name is authorized by this guide**. Stop if this cannot be demonstrated.
3. Verify backup coverage and a restore test for all data potentially affected, target identities, capacity, rollback, privilege and service impact. Review `enable_cinder_backup` and `cinder_backup_driver`: the installed lab defaults enable backup and select Ceph when backup is enabled, but this lab has no Ceph. Explicitly disable backup or configure a supported, separately provisioned backup target; do not confuse snapshots with off-host backups.
4. Only after separate change approval, back up configuration securely, change the approved inventory and globals, validate configuration, run release-matched Kolla prechecks, then deploy using the existing privileged execution procedure. Stop on any failure; prechecks alone do not validate data access. No commands to alter existing LVM are supplied while the design is unresolved.
5. With authorized disposable resources, check `openstack volume service list` and `openstack volume pool list --detail`; verify the expected services are up and each intended pool is present. Create a small test volume on **each** intended backend using a verified backend-specific volume type, wait for `available`, attach to the approved test VM, confirm the guest detects the correct disk without formatting an existing disk, detach, verify `available`, and delete only the test volume. Record backend, volume ID, VM ID and results. Registration alone is not a successful I/O test. Test backup/restore separately if configured.

**Completion gate:** all approved backends have passed create/attach/guest visibility/detach tests and documented recovery tests; otherwise mark the untested portions DEFERRED. Continue to Step 13 without Cinder if necessary.

## Step 13 — HA failure testing and recovery

**Goal.** Measure the behavior of a specific redundant component under a controlled failure, rather than assuming three nodes or a VIP make every service HA. **Status: no failover or recovery test is established by this document.** Storage failover is explicitly not demonstrated.

**Where / prerequisites.** Management VM and a user-visible terminal, with console/out-of-band access to all nodes, verified current backup/restore coverage, a disposable workload, a maintenance window, a named operator and separately approved exact target, action and rollback. Confirm no important workload depends on the test. Whole-node shutdown is not a default test and Galera/RabbitMQ must not be subjected to generic service restarts.

**Action — read-only baseline:** With a configured authenticated cloud name and without printing the token, run these on the management VM; if authentication is unavailable, stop and mark API checks BLOCKED:

```bash
openstack --os-cloud kolla-admin server list --all-projects
openstack --os-cloud kolla-admin compute service list
openstack --os-cloud kolla-admin network agent list
```

The cloud name is an example: check the names in your protected local `clouds.yaml` privately and use the actual name. Record expected node identities, each service's `up`/`enabled` state and the disposable VM's status. Inspect the [existing shutdown and recovery guide](https://github.com/DinosaurKeys/openstack-lab/blob/main/docs/SHUTDOWN-AND-RECOVERY.md) and its scripts before relying on them. Independently verify every configured Galera member's current Primary/Synced/ready status and RabbitMQ membership and partitions; a running container, reachable VIP or a single successful API request is not quorum. Do not print credentials or raw logs.

**Proposed controlled test — only after approval:** select one non-database, non-broker service with independently confirmed redundant peers; verify which backend the request reaches and capture baseline direct-backend and VIP responses. Write down the exact failure injection, maximum outage, stop conditions, restoration command and owner. Inject only that one failure, observe client errors, timing and surviving backend identity, restore exactly what was changed, then repeat the baseline and exercise the disposable workload. Stop immediately if quorum, other peers or recovery is uncertain; do not proceed to another fault. A test must not imply volume failover or full power-cycle recovery.

**Result / stop:** PASS requires recorded before/during/after observations, a healthy restored component and an independently successful workload check; record duration and scope. FAIL if any assertion fails; BLOCKED if credentials, member status or console/backup prerequisites are missing; NOT TESTED if no fault was injected. Do not run database recovery unless a separately diagnosed complete outage and exact recovery action are approved. Follow the linked procedure only after verifying it matches the current deployment.

## Step 14 — Terraform sandbox

**Goal.** Describe a disposable cloud resource in code, review a plan, then compare actual state. Terraform state is sensitive and is not a backup of the cloud.

**Where / prerequisites.** Management VM, after a manual VM/network test has passed, a separately authorized sandbox project, restricted application credential and quotas. Use [HashiCorp's install instructions](https://developer.hashicorp.com/terraform/install) and the [OpenStack provider documentation](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs). Installing software, creating credentials and any cloud resource are changes, not read-only checks.

**Action — read-only checks:**

```bash
terraform version
openstack --os-cloud kolla-admin token issue -c expires -f value
```

Use the real configured cloud name rather than the example; the second command requests a token but displays only its expiry, not the token itself. If either command or authentication fails, mark the step BLOCKED; do not print credentials to troubleshoot.

**Implementation — use only a separately approved sandbox project:**

1. Create a protected sandbox application credential using the cloud's approved procedure. Put it in a private `clouds.yaml` entry named `lab-sandbox`; never commit or display it. Choose and protect the state location. The following uses a local state file in a private directory, not a public Git repository. Do not point this exercise at an existing shared network.
2. On the management VM, create a new working directory. If the command says it exists, stop and inspect rather than using or overwriting it:

   ```bash
   mkdir -m 700 "$HOME/terraform-openstack-sandbox"
   ```

   Only after that succeeds:

   ```bash
   cd "$HOME/terraform-openstack-sandbox"
   nano main.tf
   ```

3. Put the following **configuration, not terminal commands**, into `main.tf`. Save with Ctrl+O, Enter and Ctrl+X. It plans one new, uniquely named network with no subnet or router. The provider version is pinned for reproducibility; review and deliberately update it later, rather than silently taking a new release:

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

   Validation does not prove cloud connectivity. Stop if either fails. Then review the plan in the private terminal:

   ```bash
   terraform plan
   ```

   Confirm the sandbox project and that the plan creates exactly the intended network with no unexpected updates or deletions. Plans and state can contain sensitive values; do not paste them into chat.
5. Only with explicit approval, apply and review Terraform's fresh confirmation prompt:

   ```bash
   terraform apply
   ```

   After it succeeds, list networks in the **sandbox project**, identify the exact network ID and inspect that ID. Do not assume an exit code proves the network is usable:

   ```bash
   openstack --os-cloud lab-sandbox network list
   ```

   ```bash
   openstack --os-cloud lab-sandbox network show '<verified-network-id>'
   ```

   Replace the quoted placeholder with the actual ID; do not type the angle brackets.
6. To remove it, obtain explicit destroy approval, inspect `terraform plan -destroy`, and confirm it includes only resources owned by this sandbox state. Run `terraform destroy` only after review; verify the exact network ID is absent. Never destroy shared networks or resources created outside this exercise.

Do not commit `.terraform/`, state files, plans, `clouds.yaml` or credentials. A provider lock file is distinct from these secrets but still review it before committing. This recipe's syntax can be reviewed without creating resources; do not claim a live apply or destroy unless it actually succeeded.

## Step 15 — Prometheus and Grafana

**Goal.** Prometheus collects metrics; Grafana displays them. A dashboard is useful only when its data source and scrape targets actually work. **Status: deployment and monitoring validation deferred.**

**Where / prerequisites.** Management VM and intended monitoring hosts; stable core cloud, explicitly chosen real inventory host(s), adequate RAM/disk, retention and access plan, secure credentials and a separately approved change. Do not assume the sample monitoring host exists. Keep endpoints on the management network or use an approved authenticated encrypted access path.

**Action — read-only discovery on the management VM:**

```bash
source "$HOME/.venvs/kolla-2026.2/bin/activate"
ansible-inventory -i /etc/kolla/multinode --graph monitoring
ansible-inventory -i /etc/kolla/multinode --graph prometheus
ansible-inventory -i /etc/kolla/multinode --graph grafana
```

Review the installed release's `roles/prometheus/defaults/main.yml`, prechecks, relevant service inventory children, and nonsecret monitoring flags in globals; do not dump passwords. On each intended host inspect `free -h`, `df -h` and current load (`uptime`); compare available capacity and retention against the proposed workload rather than assuming idle services free memory. No node selection or capacity decision is made by this guide.

**Implementation sequence — after approval:** choose host(s), failure and retention expectations, access restrictions, exporter scope and rollback; securely back up configuration. Add real host aliases to the correct release-matched inventory groups without damaging child relationships. Review and set `enable_prometheus: true` and `enable_grafana: true` in the existing globals rather than creating duplicate keys. Validate the YAML and effective placement. Use the matching Kolla environment and approved privilege mode; run release-matched `kolla-ansible prechecks -i /etc/kolla/multinode` (add `--use-test-images` only for the approved test-image configuration). Stop on failure. After reviewing impact, run the approved Kolla deploy, not an unreviewed host-limited operation. Kolla output can contain secrets.

**Validation gate:** through the approved authenticated path, confirm Grafana login, Prometheus data source connectivity, scrape-target status for each expected exporter and fresh timestamped host metrics in a dashboard. Confirm access is denied on unintended networks; test alerts only if alerting is configured. A running container or login page alone is not a metrics test. If authentication, a target, or fresh data is missing, report the exact portion BLOCKED or FAIL and stop before relying on alerts.

**References:** [Kolla 2026.2 operating guidance](https://docs.openstack.org/kolla-ansible/2026.2/user/operating-kolla.html), [Prometheus querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/), and [Grafana data sources](https://grafana.com/docs/grafana/latest/datasources/). Review installed role defaults in addition to upstream documentation; availability and defaults can differ by release.
