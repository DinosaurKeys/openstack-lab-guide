# Deploy OpenStack with Kolla-Ansible

After completing the fresh Ubuntu, SSH, Kolla-Ansible and network setup, run these steps in order on the **management VM** as your normal administrator, unless a step says otherwise. The fresh-build example calls it `manager`; Joe's current management VM is named `r2d2`. Both names refer to the same machine. The physical nodes are `node1` (`192.168.0.200`), `node2` (`192.168.0.201`), and `node3` (`192.168.0.202`). Verify these addresses against your own reserved address plan before targeting them.

At the start of every new R2D2 terminal, run the Step 00 path contract so `$KOLLA_VENV`, `$KOLLA_CONFIG_PATH` and `$KOLLA_INVENTORY` are defined. Do not substitute `/etc/kolla` or the historical virtual environment.

**Existing cloud? Stop.** This is a fresh-install procedure, not permission to overwrite an existing inventory, configuration, passwords, services, or data. First review the differences and the backup and recovery plan, then obtain approval for the exact changes. Never put passwords, tokens, private keys, `clouds.yaml`, or raw Kolla output in Git or chat.

## Step 5: Build and check the multinode inventory

**Goal:** Put all three nodes in each core role and preserve the installed release's service mappings. **Where:** the management VM. The starting `$KOLLA_CONFIG_PATH/multinode` must be the copy of the `stable/2026.2` inventory installed in the previous step.

Activate the environment and check that Ansible comes from it:

```bash
source "$KOLLA_VENV/bin/activate"
command -v ansible-playbook
```

**Expected:** the path ends in `.venvs/kolla-fresh/bin/ansible-playbook`. Stop if activation fails or the path differs. Repeat this in each new terminal.

Before editing, check whether the backup already exists. If it does, stop and inspect it instead of overwriting it:

```bash
if [ -e "$KOLLA_CONFIG_PATH/multinode.pre-step5" ]; then
  echo 'STOP: inventory backup already exists.'
else
  cp -a -- "$KOLLA_CONFIG_PATH/multinode" "$KOLLA_CONFIG_PATH/multinode.pre-step5" &&
    cmp -s -- "$KOLLA_CONFIG_PATH/multinode" "$KOLLA_CONFIG_PATH/multinode.pre-step5" &&
    echo 'Inventory backup verified.'
fi
```

Continue only after you see `Inventory backup verified.` Edit `$KOLLA_CONFIG_PATH/multinode` with `nano $KOLLA_CONFIG_PATH/multinode`. Replace the **existing** host entries in these sections, not the whole file:

```ini
[control]
node1 ansible_host=192.168.0.200
node2 ansible_host=192.168.0.201
node3 ansible_host=192.168.0.202

[network]
node1
node2
node3

[compute]
node1
node2
node3

[monitoring]
# Empty until monitoring hosts are selected.

[storage]
# Empty until a storage design and hosts are selected.
```

Replace or remove **all** active upstream example hosts, including `control01`, `network01`, `compute01`, `monitoring01`, and `storage01`. Leave `[deployment]` and all child groups and their mappings intact, including `[loadbalancer:children]` → `network`. Do not rename hyphenated groups.

### Find the three `[all:vars]` values

These values come from the SSH and privilege checks in Step 2. They describe how Ansible connects from R2D2 to the physical nodes:

| Variable | Meaning | Where to get it |
|---|---|---|
| `ansible_user` | The normal administrator account on the nodes; it is not necessarily the same as the inventory hostname. | The account used successfully in the Step 2 SSH test. For this lab, it is `joe`. |
| `ansible_private_key_file` | The private SSH key stored on R2D2. This is an absolute path on the management VM, not a path on the nodes. | The key created or selected in Step 2. For this lab, verify `/home/joe/.ssh/openstack_lab_ed25519` exists. Never copy or display its contents. |
| `ansible_become` | Tells Ansible to use `sudo` for privileged tasks. `true` does not contain a password. | Set it to `true` after `sudo -n id -u` returns `0` on every node. |

Run this read-only verification on R2D2 before editing the inventory. It does not display the private key or any password:

```bash
KEY="$HOME/.ssh/openstack_lab_ed25519"
NODE_USER="joe"

if [ ! -f "$KEY" ]; then
  printf 'STOP: SSH private key not found: %s\n' "$KEY"
  exit 1
fi

stat -c 'SSH key path=%n owner=%U mode=%a' "$KEY"

for address in 192.168.0.200 192.168.0.201 192.168.0.202; do
  printf '\n--- %s ---\n' "$address"
  ssh -o BatchMode=yes -o ConnectTimeout=10 \
    -i "$KEY" "$NODE_USER@$address" \
    'printf "remote_user=%s\n" "$(id -un)"; printf "sudo_uid="; sudo -n id -u'
done
```

**Pass when:** the key exists on R2D2, each node reports `remote_user=joe`, and each node reports `sudo_uid=0`. Stop if SSH or non-interactive sudo fails; do not guess the values.

If the `[all:vars]` section is missing from the release-provided inventory, add it at the bottom of `$KOLLA_CONFIG_PATH/multinode`. For this lab, the verified section is:

```ini
[all:vars]
ansible_user=joe
ansible_private_key_file=/home/joe/.ssh/openstack_lab_ed25519
ansible_become=true
```

Do not put a sudo password in the inventory. If a deployment requires an interactive sudo password, enter it only through Ansible's secure prompt. Preserve any other needed variables, avoid duplicate active assignments, and save the file. Leave storage and monitoring disabled in globals until independently planned.

Parse the inventory, then inspect the graph:

```bash
ansible-inventory -i $KOLLA_CONFIG_PATH/multinode --graph
```

**Expected:** exactly the three named nodes in each of `control`, `network`, and `compute`; empty `storage` and `monitoring`; `localhost` in `deployment`; no sample hosts. Check the inherited child groups as well. Warnings about hyphenated names alone are not an error. Stop and correct any unexpected membership before running a node command.

```bash
ansible -i $KOLLA_CONFIG_PATH/multinode control -m ping -e '{"ansible_become": false}'
```

**Expected:** `pong` from all three nodes, with no failed or unreachable hosts. This checks SSH and Python, but not sudo. Verify SSH host keys through a trusted channel; do not disable host-key checking. Stop on a failure.

## Step 6: Verify names and time

**Goal:** Check that all four machines agree on node names and have synchronized clocks in the chosen `Europe/Bucharest` timezone. **Where:** the management VM. The following commands are read-only; they target the three nodes and the `deployment` host.

```bash
ansible -i $KOLLA_CONFIG_PATH/multinode 'control:deployment' \
  -m ansible.builtin.command -a 'timedatectl show -p Timezone -p NTPSynchronized' \
  -e '{"ansible_become": false}'
```

**Expected:** every host reports `Timezone=Europe/Bucharest`, `NTPSynchronized=yes`, and no failed/unreachable result. A `CHANGED` label from the command module does not mean this query changed the clock. Check the supported time-service alternatives:

```bash
ansible -i $KOLLA_CONFIG_PATH/multinode 'control:deployment' \
  -m ansible.builtin.shell \
  -a 'for s in chrony chronyd ntp ntpd systemd-timesyncd; do systemctl is-active --quiet "$s" && echo "$s active"; done; true' \
  -e '{"ansible_become": false}'
```

**Expected:** at least one supported service is active on each host; inactive or absent alternatives are normal. Then check name resolution:

```bash
ansible -i $KOLLA_CONFIG_PATH/multinode 'control:deployment' \
  -m ansible.builtin.command -a 'getent hosts node1 node2 node3' \
  -e '{"ansible_become": false}'
```

**Expected:** each machine resolves `node1`, `node2`, and `node3` to `192.168.0.200`, `.201`, and `.202` respectively, not loopback. Stop if any check fails or an address differs. Inspect host mappings, the service and upstream time connectivity before making changes. Do not install a second time daemon merely because alternatives are inactive. Any timezone or service correction requires separate authorization. Setting a timezone is not the same as synchronizing the clock.

## Step 7: Configure globals and protect passwords

**Goal:** Configure this lab's networking and create fresh service credentials. **Where:** the management VM only; editing these files does not deploy services.

First, confirm on **every node** that `mgmt0` is the intended management interface and `provider0` is the separate connected provider interface without a configured provider-subnet address or default route. Confirm `192.168.0.150` is on the management subnet, reserved outside DHCP and other allocations, and not assigned to a device. A failed ping alone does not establish availability. Stop if the plan, cable, route, reservation or address ownership is uncertain.

Back up globals, taking care not to overwrite an earlier backup:

```bash
if [ -e "$KOLLA_CONFIG_PATH/globals.yml.pre-step7" ]; then
  echo 'STOP: globals backup already exists.'
else
  cp -a -- "$KOLLA_CONFIG_PATH/globals.yml" "$KOLLA_CONFIG_PATH/globals.yml.pre-step7" &&
    cmp -s -- "$KOLLA_CONFIG_PATH/globals.yml" "$KOLLA_CONFIG_PATH/globals.yml.pre-step7" &&
    echo 'Globals backup verified.'
fi
```

Continue only after confirmation. Open the file with `nano $KOLLA_CONFIG_PATH/globals.yml`, then uncomment or change the existing keys rather than adding duplicates:

```yaml
kolla_base_distro: "ubuntu"
network_interface: "mgmt0"
neutron_external_interface: "provider0"
enable_neutron_provider_networks: true
kolla_internal_vip_address: "192.168.0.150"
enable_heat: false
enable_cinder: false
enable_prometheus: false
enable_grafana: false
```

The VIP is shared by endpoints; it is not a node address. The provider interface connects Neutron to the planned network, but a link alone does not establish upstream connectivity. Optional storage and monitoring remain disabled for the base cloud. Save and check YAML syntax **and duplicate keys** without printing values:

```bash
python - <<'PY'
import os
import yaml
class UniqueLoader(yaml.SafeLoader):
    pass
def mapping(loader, node, deep=False):
    result = {}
    for key_node, value_node in node.value:
        key = loader.construct_object(key_node, deep=deep)
        if key in result:
            raise ValueError('Duplicate YAML key: ' + str(key))
        result[key] = loader.construct_object(value_node, deep=deep)
    return result
UniqueLoader.add_constructor(yaml.resolver.BaseResolver.DEFAULT_MAPPING_TAG, mapping)
with open(os.path.join(os.environ['KOLLA_CONFIG_PATH'], 'globals.yml'), encoding='utf-8') as f:
    yaml.load(f, Loader=UniqueLoader)
print('YAML syntax and unique keys OK')
PY
```

Stop on any error and fix it. For a **new installation only**, make a one-time verified backup before generating missing passwords. Do not regenerate credentials for an existing deployment:

```bash
if [ -e "$KOLLA_CONFIG_PATH/passwords.yml.pre-kolla-genpwd" ]; then
  printf '%s\n' 'STOP: passwords backup already exists; inspect it before continuing.'
else
  cp -a "$KOLLA_CONFIG_PATH/passwords.yml" \
    "$KOLLA_CONFIG_PATH/passwords.yml.pre-kolla-genpwd" &&
  cmp -s "$KOLLA_CONFIG_PATH/passwords.yml" \
    "$KOLLA_CONFIG_PATH/passwords.yml.pre-kolla-genpwd" &&
  printf '%s\n' 'Passwords backup verified.'
fi
```

Then generate the credentials using the **explicit fresh configuration path**:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
source "$KOLLA_VENV/bin/activate"
umask 077
kolla-genpwd -p "$KOLLA_CONFIG_PATH/passwords.yml" &&
  chmod 600 "$KOLLA_CONFIG_PATH/passwords.yml"
```

The `-p` option is required here because `kolla-genpwd` otherwise defaults to `/etc/kolla/passwords.yml`; `KOLLA_CONFIG_PATH` alone does not select the password file for this command. The backup allows restoration of the previous file if needed. **Expected:** the command succeeds and the file is mode 600 (`stat -c '%a %n' "$KOLLA_CONFIG_PATH/passwords.yml"`). Keep the backup private. Do not print the file or blindly regenerate credentials for an existing deployment.

## Step 8: Verify sudo, bootstrap and precheck

**Goal:** Confirm root access, install host dependencies, and then test readiness. **Where:** the management VM. Bootstrap makes real package and system-configuration changes on nodes. Review the [2026.2 bootstrap reference](https://docs.openstack.org/kolla-ansible/2026.2/reference/deployment-and-bootstrapping/bootstrap-servers.html), target list, backups and console access first. Do not use this fresh-install procedure on an existing cloud without a separately reviewed plan.

SSH authentication and sudo are separate. Test the lab's passwordless mode on all nodes **and** the local deployment host:

```bash
ANSIBLE_BECOME_ASK_PASS=false ansible -i $KOLLA_CONFIG_PATH/multinode 'control:deployment' \
  -m ansible.builtin.command -a 'id -u' \
  -e '{"ansible_become": true, "ansible_become_flags": "-H -n"}'
```

**Expected:** all four report `0` and `rc=0`, with no failed/unreachable hosts. The command module uses the label `CHANGED`; it does not mean this test changed the configuration. If any fail, stop. Passwordless become has not been established. If interactive sudo is intended instead, test in a user-visible terminal:

```bash
ansible -i $KOLLA_CONFIG_PATH/multinode 'control:deployment' \
  -m ansible.builtin.command -a 'id -u' \
  -e '{"ansible_become": true}' --ask-become-pass
```

Enter the password only at the local prompt. The single prompt must work on every selected host; if passwords differ, stop and arrange a secure per-host credential method. Never store a password in inventory, shell commands, environment variables, or chat. Unrestricted passwordless sudo gives processes using that account root access; do not add or widen sudo policy without explicit approval.

For verified passwordless sudo, set the mode in this terminal:

```bash
export ANSIBLE_BECOME_ASK_PASS=false
```

For the verified interactive alternative, use `export ANSIBLE_BECOME_ASK_PASS=true` instead; `true` is a literal option, **not a password**. Kolla's CLI invokes Ansible and honors this setting even though it does not expose `-K`. Set it again in each new terminal, and keep it for `validate-config` and later `post-deploy` too.

**Firewall safety:** the release's baremetal bootstrap can disable UFW and set an allow policy. The extra variable below skips that firewall-management task; it does not enable UFW, open ports, or prove current rules permit traffic. Inspect existing firewall state and rules on each node, and review service-specific ports, traffic directions and trusted sources before applying any default-deny policy. Keep console recovery available.

```bash
kolla-ansible bootstrap-servers \
  -i "$KOLLA_CONFIG_PATH/multinode" \
  -e disable_firewall=false
```

**Expected:** successful completion and no failed/unreachable hosts. If it fails, stop and inspect the failed task and each host before retrying. Some changes may already have occurred. Do not proceed to prechecks until bootstrap succeeds.

For the lab's default `quay.io/openstack.kolla` testing images, explicitly acknowledge test images:

```bash
kolla-ansible prechecks \
  -i "$KOLLA_CONFIG_PATH/multinode" \
  --use-test-images
```

**Expected:** no failed/unreachable hosts. Stop, resolve the issue, and rerun this gate on failure. The flag acknowledges testing images; it neither deploys services nor proves image availability or a working cloud. For another registry, review its settings rather than blindly using the flag. Kolla output can expose secrets even in skipped or failed tasks; do not share raw output. If you need assistance, provide only carefully sanitized failure details and recap.

### 101 explanation — prepare first, deploy second

The two commands have different jobs:

```text
R2D2 → bootstrap-servers → prepare Ubuntu nodes
     → prechecks         → verify readiness
     → deploy            → install and start OpenStack services
```

**`bootstrap-servers` means:** prepare the three physical Ubuntu nodes. It can install host prerequisites, prepare Docker and Kolla directories, and apply required host settings. It changes the node operating systems but does not deploy the OpenStack control plane.

**`-i "$KOLLA_CONFIG_PATH/multinode"` means:** use the Kolla inventory to decide which machines are `control`, `network`, `compute`, and other roles.

**`-e disable_firewall=false` means:** do not let the bootstrap firewall task disable or reconfigure the host firewall. It does not enable UFW, open ports, or prove that required traffic is allowed.

**`deploy` means:** install and start the configured OpenStack services. It reads the inventory, `globals.yml`, and generated `passwords.yml`, then creates and configures services such as MariaDB, RabbitMQ, HAProxy, Keystone, Glance, Nova, Neutron and Horizon.

A VMware analogy: **bootstrap prepares the ESXi hosts; deploy installs and starts the workloads.** Bootstrap and prechecks must pass before deployment. Deployment is a real infrastructure change, not a dry run.

## Step 9: Deploy and validate

**Goal:** Install the configured services and validate the generated configuration. **Where:** the management VM with the same venv and tested sudo mode active. This is a real host and cloud change, not a dry run. Continue only after Steps 5–8 pass, the exact targets are reviewed, and the fresh deployment is authorized.

```bash
kolla-ansible deploy \
  -i "$KOLLA_CONFIG_PATH/multinode"
```

**Expected:** the command finishes successfully and its recap has no failed or unreachable hosts. Image downloads and initialization can take time. If it fails, stop. A partial deployment may exist. Investigate locally without posting logs or replacing credentials, and do not retry blindly.

Run the following only after deploy succeeds:

```bash
kolla-ansible validate-config -i $KOLLA_CONFIG_PATH/multinode
```

**Expected:** successful completion and a recap with no failed or unreachable hosts. Stop and investigate failures before proceeding.

To check processes initially, sign in to **each physical node** separately and run:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
```

Compare the running containers and any reported health states against the enabled services and their inventory placement. Before continuing, investigate any expected containers that are missing, restarting or unhealthy. A running container alone does not prove its API or cluster is healthy. Return to the management VM afterward.

Neither a successful recap nor configuration validation proves Keystone authentication works or a VM can boot. In the next cloud-verification step, check authenticated API access and a disposable VM separately. Report each result you actually observe. Do not claim a live deployment or verification based on this guide alone.

## Optional: Disable Heat before a fresh deployment

Heat provides orchestration and is not needed for the base lab. On the management VM, set `enable_heat: false` in the existing `$KOLLA_CONFIG_PATH/globals.yml` **before Step 8 and the first deployment**; Step 7 includes the setting. Edit the existing key rather than adding a duplicate, and run the Step 7 YAML and duplicate-key check. Verify the active setting without exposing other configuration:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
grep -nE '^[[:space:]]*enable_heat[[:space:]]*:' "$KOLLA_CONFIG_PATH/globals.yml"
```

**Pass when:** exactly one active line says `enable_heat: false`. Deploy normally only after the previous checks pass. Heat APIs and stack orchestration will not be available.

**Already deployed?** Changing this setting alone does not stop or remove existing Heat containers, stacks, endpoints, or RabbitMQ queues. Inspect the deployed services and use a separately reviewed, approved removal plan; do not run the fresh-install procedure against an existing cloud. For queue inspection and carefully targeted deletion, see [RabbitMQ queue cleanup](RABBITMQ-QUEUE-CLEANUP.md).
