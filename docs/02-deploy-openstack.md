# Deploy OpenStack with Kolla-Ansible

After completing the fresh Ubuntu, SSH, Kolla-Ansible and network setup, run these steps in order on the **management VM** as your normal administrator, unless a step says otherwise. The fresh-build example calls it `manager`; Joe's current management VM is named `r2d2`. Both names refer to the same machine. The physical nodes are `node1` (`192.168.0.200`), `node2` (`192.168.0.201`), and `node3` (`192.168.0.202`). Verify these addresses against your own reserved address plan before targeting them.

**Existing cloud? Stop.** This is a fresh-install procedure, not permission to overwrite an existing inventory, configuration, passwords, services, or data. First review the differences and the backup and recovery plan, then obtain approval for the exact changes. Never put passwords, tokens, private keys, `clouds.yaml`, or raw Kolla output in Git or chat.

## Step 5: Build and check the multinode inventory

**Goal:** Put all three nodes in each core role and preserve the installed release's service mappings. **Where:** the management VM. The starting `/etc/kolla/multinode` must be the copy of the `stable/2026.2` inventory installed in the previous step.

Activate the environment and check that Ansible comes from it:

```bash
source "$HOME/.venvs/kolla-2026.2/bin/activate"
command -v ansible-playbook
```

**Expected:** the path ends in `.venvs/kolla-2026.2/bin/ansible-playbook`. Stop if activation fails or the path differs. Repeat this in each new terminal.

Before editing, check whether the backup already exists. If it does, stop and inspect it instead of overwriting it:

```bash
if [ -e /etc/kolla/multinode.pre-step5 ]; then
  echo 'STOP: inventory backup already exists.'
else
  cp -a /etc/kolla/multinode /etc/kolla/multinode.pre-step5 &&
    cmp -s /etc/kolla/multinode /etc/kolla/multinode.pre-step5 &&
    echo 'Inventory backup verified.'
fi
```

Continue only after you see `Inventory backup verified.` Edit `/etc/kolla/multinode` with `nano /etc/kolla/multinode`. Replace the **existing** host entries in these sections, not the whole file:

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

Replace or remove **all** active upstream example hosts, including `control01`, `network01`, `compute01`, `monitoring01`, and `storage01`. Leave `[deployment]` and all child groups and their mappings intact, including `[loadbalancer:children]` → `network`. Do not rename hyphenated groups. Configure the actual SSH account and key from the earlier SSH setup in the existing `[all:vars]` section, adding it only if absent:

```ini
[all:vars]
ansible_user=REPLACE_WITH_NODE_ADMIN_USER
ansible_private_key_file=REPLACE_WITH_ABSOLUTE_PRIVATE_KEY_PATH
ansible_become=True
```

Replace both placeholders with the verified values. The key path is on the management VM. Never put a sudo password here. Preserve any other needed variables, avoid duplicate active assignments, and save the file. Leave storage and monitoring disabled in globals until independently planned.

Parse the inventory, then inspect the graph:

```bash
ansible-inventory -i /etc/kolla/multinode --graph
```

**Expected:** exactly the three named nodes in each of `control`, `network`, and `compute`; empty `storage` and `monitoring`; `localhost` in `deployment`; no sample hosts. Check the inherited child groups as well. Warnings about hyphenated names alone are not an error. Stop and correct any unexpected membership before running a node command.

```bash
ansible -i /etc/kolla/multinode control -m ping -e '{"ansible_become": false}'
```

**Expected:** `pong` from all three nodes, with no failed or unreachable hosts. This checks SSH and Python, but not sudo. Verify SSH host keys through a trusted channel; do not disable host-key checking. Stop on a failure.

## Step 6: Verify names and time

**Goal:** Check that all four machines agree on node names and have synchronized clocks in the chosen `Europe/Bucharest` timezone. **Where:** the management VM. The following commands are read-only; they target the three nodes and the `deployment` host.

```bash
ansible -i /etc/kolla/multinode 'control:deployment' \
  -m ansible.builtin.command -a 'timedatectl show -p Timezone -p NTPSynchronized' \
  -e '{"ansible_become": false}'
```

**Expected:** every host reports `Timezone=Europe/Bucharest`, `NTPSynchronized=yes`, and no failed/unreachable result. A `CHANGED` label from the command module does not mean this query changed the clock. Check the supported time-service alternatives:

```bash
ansible -i /etc/kolla/multinode 'control:deployment' \
  -m ansible.builtin.shell \
  -a 'for s in chrony chronyd ntp ntpd systemd-timesyncd; do systemctl is-active --quiet "$s" && echo "$s active"; done; true' \
  -e '{"ansible_become": false}'
```

**Expected:** at least one supported service is active on each host; inactive or absent alternatives are normal. Then check name resolution:

```bash
ansible -i /etc/kolla/multinode 'control:deployment' \
  -m ansible.builtin.command -a 'getent hosts node1 node2 node3' \
  -e '{"ansible_become": false}'
```

**Expected:** each machine resolves `node1`, `node2`, and `node3` to `192.168.0.200`, `.201`, and `.202` respectively, not loopback. Stop if any check fails or an address differs. Inspect host mappings, the service and upstream time connectivity before making changes. Do not install a second time daemon merely because alternatives are inactive. Any timezone or service correction requires separate authorization. Setting a timezone is not the same as synchronizing the clock.

## Step 7: Configure globals and protect passwords

**Goal:** Configure this lab's networking and create fresh service credentials. **Where:** the management VM only; editing these files does not deploy services.

First, confirm on **every node** that `mgmt0` is the intended management interface and `provider0` is the separate connected provider interface without a configured provider-subnet address or default route. Confirm `192.168.0.150` is on the management subnet, reserved outside DHCP and other allocations, and not assigned to a device. A failed ping alone does not establish availability. Stop if the plan, cable, route, reservation or address ownership is uncertain.

Back up globals, taking care not to overwrite an earlier backup:

```bash
if [ -e /etc/kolla/globals.yml.pre-step7 ]; then
  echo 'STOP: globals backup already exists.'
else
  cp -a /etc/kolla/globals.yml /etc/kolla/globals.yml.pre-step7 &&
    cmp -s /etc/kolla/globals.yml /etc/kolla/globals.yml.pre-step7 &&
    echo 'Globals backup verified.'
fi
```

Continue only after confirmation. Open the file with `nano /etc/kolla/globals.yml`, then uncomment or change the existing keys rather than adding duplicates:

```yaml
kolla_base_distro: "ubuntu"
network_interface: "mgmt0"
neutron_external_interface: "provider0"
enable_neutron_provider_networks: true
kolla_internal_vip_address: "192.168.0.150"
enable_cinder: false
enable_prometheus: false
enable_grafana: false
```

The VIP is shared by endpoints; it is not a node address. The provider interface connects Neutron to the planned network, but a link alone does not establish upstream connectivity. Optional storage and monitoring remain disabled for the base cloud. Save and check YAML syntax **and duplicate keys** without printing values:

```bash
python - <<'PY'
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
with open('/etc/kolla/globals.yml', encoding='utf-8') as f:
    yaml.load(f, Loader=UniqueLoader)
print('YAML syntax and unique keys OK')
PY
```

Stop on any error and fix it. For a **new installation only**, generate missing passwords in the existing release-provided `/etc/kolla/passwords.yml`:

```bash
umask 077
kolla-genpwd && chmod 600 /etc/kolla/passwords.yml
```

**Expected:** command succeeds and the file is mode 600 (`stat -c '%a %n' /etc/kolla/passwords.yml`). Keep a protected, private backup. Do not print the file or blindly regenerate credentials for an existing deployment.

## Step 8: Verify sudo, bootstrap and precheck

**Goal:** Confirm root access, install host dependencies, and then test readiness. **Where:** the management VM. Bootstrap makes real package and system-configuration changes on nodes. Review the [2026.2 bootstrap reference](https://docs.openstack.org/kolla-ansible/2026.2/reference/deployment-and-bootstrapping/bootstrap-servers.html), target list, backups and console access first. Do not use this fresh-install procedure on an existing cloud without a separately reviewed plan.

SSH authentication and sudo are separate. Test the lab's passwordless mode on all nodes **and** the local deployment host:

```bash
ANSIBLE_BECOME_ASK_PASS=false ansible -i /etc/kolla/multinode 'control:deployment' \
  -m ansible.builtin.command -a 'id -u' \
  -e '{"ansible_become": true, "ansible_become_flags": "-H -n"}'
```

**Expected:** all four report `0` and `rc=0`, with no failed/unreachable hosts. The command module uses the label `CHANGED`; it does not mean this test changed the configuration. If any fail, stop. Passwordless become has not been established. If interactive sudo is intended instead, test in a user-visible terminal:

```bash
ansible -i /etc/kolla/multinode 'control:deployment' \
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
kolla-ansible bootstrap-servers -i /etc/kolla/multinode -e disable_firewall=false
```

**Expected:** successful completion and no failed/unreachable hosts. If it fails, stop and inspect the failed task and each host before retrying. Some changes may already have occurred. Do not proceed to prechecks until bootstrap succeeds.

For the lab's default `quay.io/openstack.kolla` testing images, explicitly acknowledge test images:

```bash
kolla-ansible prechecks -i /etc/kolla/multinode --use-test-images
```

**Expected:** no failed/unreachable hosts. Stop, resolve the issue, and rerun this gate on failure. The flag acknowledges testing images; it neither deploys services nor proves image availability or a working cloud. For another registry, review its settings rather than blindly using the flag. Kolla output can expose secrets even in skipped or failed tasks; do not share raw output. If you need assistance, provide only carefully sanitized failure details and recap.

## Step 9: Deploy and validate

**Goal:** Install the configured services and validate the generated configuration. **Where:** the management VM with the same venv and tested sudo mode active. This is a real host and cloud change, not a dry run. Continue only after Steps 5–8 pass, the exact targets are reviewed, and the fresh deployment is authorized.

```bash
kolla-ansible deploy -i /etc/kolla/multinode
```

**Expected:** the command finishes successfully and its recap has no failed or unreachable hosts. Image downloads and initialization can take time. If it fails, stop. A partial deployment may exist. Investigate locally without posting logs or replacing credentials, and do not retry blindly.

Run the following only after deploy succeeds:

```bash
kolla-ansible validate-config -i /etc/kolla/multinode
```

**Expected:** successful completion and a recap with no failed or unreachable hosts. Stop and investigate failures before proceeding.

To check processes initially, sign in to **each physical node** separately and run:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
```

Compare the running containers and any reported health states against the enabled services and their inventory placement. Before continuing, investigate any expected containers that are missing, restarting or unhealthy. A running container alone does not prove its API or cluster is healthy. Return to the management VM afterward.

Neither a successful recap nor configuration validation proves Keystone authentication works or a VM can boot. In the next cloud-verification step, check authenticated API access and a disposable VM separately. Report each result you actually observe. Do not claim a live deployment or verification based on this guide alone.
