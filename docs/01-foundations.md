# Foundations: Steps 1–4

> **Private draft — fresh install only.** These instructions are for a newly installed management VM and three newly installed physical nodes. The current cloud is live: **do not run installation, package, networking, or configuration changes against it**. An existing-cloud investigation is a separate, read-only task. Nothing here authorizes a rebuild or disk initialization.

Kolla-Ansible runs from a management VM and uses SSH and Ansible to configure the physical nodes. Think of the VM as an automation workstation, not a fourth OpenStack node. Each of the three nodes will carry control, network and compute roles; three control nodes do not automatically make every service or storage backend highly available.

| System | Planned management address | Purpose |
|---|---|---|
| `manager` VM | Unique, reachable address | Deployment workstation; it may use a routed VMware NAT network if it can reach the nodes. |
| `node1` | `192.168.0.200/24` | control, network, compute |
| `node2` | `192.168.0.201/24` | control, network, compute |
| `node3` | `192.168.0.202/24` | control, network, compute |

The home LAN is `192.168.0.0/24`, with gateway `192.168.0.1`. The planned shared API VIP is `192.168.0.150`; the planned Neutron external allocation pool is `192.168.0.151–192.168.0.199`. These ranges are reserved outside DHCP; check current router leases, static allocations, and network-owner records for actual conflicts before assigning **any** of them. An unanswered ping is not proof an address is unused. The VIP and floating IPs are not assigned to a host in these steps.

Each node's management and provider cables connect to the same home switch and the intended untagged LAN/VLAN. They are **separate physical ports**, not the same NIC under two names. `mgmt0` has the host management address and default route; `provider0` is a Layer-2-only uplink for a later Neutron external bridge. As with separate VMware uplinks, cable and VLAN placement matter; a link light does not prove upstream routing or firewall access. Record the actual MAC and cable for each role on each node. Do not bridge the ports manually. Later Neutron provider-network configuration must use this LAN, gateway and reserved pool consistently.

References: [Kolla 2026.2 quickstart](https://docs.openstack.org/kolla-ansible/2026.2/user/quickstart.html), [multinode deployment](https://docs.openstack.org/kolla-ansible/2026.2/user/multinode.html), [support matrix](https://docs.openstack.org/kolla-ansible/2026.2/user/support-matrix), [Neutron networking](https://docs.openstack.org/kolla-ansible/2026.2/reference/networking/neutron.html), [Ubuntu Server install](https://ubuntu.com/tutorials/install-ubuntu-server), and [Netplan reference](https://netplan.readthedocs.io/en/stable/netplan-yaml/).

## Step 1 — Install and identify the systems

**Goal:** Get four Ubuntu Server 24.04 systems ready without touching the existing SATA LVM pools. **Where:** VM console for `manager`, physical or out-of-band console for each node. **Prerequisites:** An independently verified backup appropriate to anything at risk, installation media, console access, confirmed NIC/cable layout, and an address-conflict check. These actions are for replacement/fresh systems only.

1. Install Ubuntu Server 24.04 LTS on the management VM. On each physical node, positively identify its **NVMe OS disk** by model, serial and capacity in the installer; select only that disk. The existing SATA disks contain LVM pools: preserve them, their partitions and metadata. Never select them for formatting or automated storage layout. If the installer cannot clearly distinguish devices or proposes changing SATA disks, stop. Installing on the selected NVMe erases its selected contents.
2. Set hostnames `manager`, `node1`, `node2`, `node3`; use the same normal sudo-capable administrator username on each. Do not enable root SSH login. Give the manager a unique address with working routed access to the nodes; it does not have to be on the same subnet. Nodes may initially use working temporary addresses until Step 4. Install only the base system; Kolla later manages OpenStack dependencies.
3. On **each system's own console**, check the identity and storage before proceeding:

```bash
hostnamectl
. /etc/os-release && printf '%s %s\n' "$PRETTY_NAME" "$VERSION_ID"
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
ip -brief address
ip route
```

On each node also inspect, without modifying, LVM metadata:

```bash
sudo pvs
sudo vgs
sudo lvs -a
```

Compare disk serials and LVM results with your records. No `pvcreate`, formatting, or Cinder provisioning is part of this guide section. Do not assume a disk is unused because it is not mounted.

4. On **each fresh system**, update packages and reboot one system at a time:

```bash
sudo apt update && sudo apt full-upgrade -y
```

If that completes successfully, run `sudo reboot`, reconnect by console, and verify the hostname and address again. These commands alter the operating system and can restart services; never use them on the live cloud under this draft's authority.

5. On each **newly installed system**, set the chosen local timezone and enable network time synchronization. These commands change host settings:

```bash
sudo timedatectl set-timezone Europe/Bucharest
sudo timedatectl set-ntp true
```

Then inspect the result:

```bash
timedatectl show -p Timezone -p NTPSynchronized
```

**Expected:** `Timezone=Europe/Bucharest` and eventually `NTPSynchronized=yes`. Synchronization can take time and requires working network and a supported time service. If it stays `no`, inspect the active time service, network and upstream time source before moving on; do not install several time daemons. Step 6 repeats the check across all four machines. Timezone controls local display, not the instant kept by the clock.

**Expected result:** Four individually identified Ubuntu 24.04 systems boot; SATA pools are still present and untouched; console and network access work. **Stop if:** disk identities or pool status are uncertain, installation offers to alter a SATA disk, any address conflicts, or a system fails to return after reboot. Resolve before Step 2.

## Step 2 — Establish SSH access

**Goal:** The manager can securely log in to all nodes. **Where:** First each system's console, then the manager terminal. **Prerequisites:** Step 1 complete, working temporary or final management addresses, a sudo-capable account and verified host identities.

On the **manager**:

```bash
sudo apt update && sudo apt install -y openssh-client openssh-server git python3 python3-venv
sudo systemctl enable --now ssh
```

On **each node**, using its console:

```bash
sudo apt update && sudo apt install -y openssh-server python3 sudo
sudo systemctl enable --now ssh
sudo systemctl is-active ssh
```

Stop on any failure. On a fresh manager, the `.ssh` directory may not exist. Check it first and create it only when absent; do not replace an existing directory or follow a symlink:

```bash
if [ -L "$HOME/.ssh" ]; then
  printf '%s\n' 'STOP: .ssh is a symlink; inspect it first.'
elif [ -d "$HOME/.ssh" ]; then
  stat -c 'Existing .ssh permissions: %a owner: %U' "$HOME/.ssh"
elif [ -e "$HOME/.ssh" ]; then
  printf '%s\n' 'STOP: .ssh exists but is not a directory.'
else
  mkdir -m 700 "$HOME/.ssh" && printf '%s\n' 'Created private .ssh directory.'
fi
```

Stop on an unexpected owner or permissions; inspect rather than blindly changing an existing directory. Then check that the dedicated key path does not already exist; never overwrite an existing key:

```bash
if [ -e "$HOME/.ssh/openstack_lab_ed25519" ] || [ -e "$HOME/.ssh/openstack_lab_ed25519.pub" ]; then
  printf '%s\n' 'Key path exists; inspect it and choose a different path or intentionally reuse it.'
else
  ssh-keygen -t ed25519 -a 64 -f "$HOME/.ssh/openstack_lab_ed25519"
fi
```

Protect the private key and its passphrase. For each node, independently check its SSH host-key fingerprint at that node's console:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On the **manager**, substitute the actual username and current address, and repeat once for each node. Verify the fingerprint shown during connection against that node's console before accepting it. Initial account passwords are entered only at the local SSH prompt, never in commands or chat:

```bash
ssh-copy-id -i "$HOME/.ssh/openstack_lab_ed25519.pub" ADMIN_USER@NODE_CURRENT_IP
```

Then test each node, replacing both placeholders each time:

```bash
ssh -i "$HOME/.ssh/openstack_lab_ed25519" ADMIN_USER@NODE_CURRENT_IP hostname
```

It must return the expected node hostname without a node account-password prompt. A key passphrase prompt is different; for multiple connections optionally load the key into a local agent with `eval "$(ssh-agent -s)"` then `ssh-add "$HOME/.ssh/openstack_lab_ed25519"`. SSH access does not prove sudo/become access; test that separately before privileged deployment. Do not disable host-key verification or share credentials.

**Expected result:** All three identity-checked SSH tests return their respective hostnames. **Stop if:** fingerprints differ, authentication fails, or the wrong host answers. Fix that node before proceeding with node-targeting operations; manager-only installation can still proceed.

## Step 3 — Install Kolla-Ansible on the manager

**Goal:** Prepare a release-matched automation environment, without deploying to nodes. **Where:** `manager` terminal as its normal administrator. **Prerequisites:** Fresh manager, internet/APT access, working sudo, no existing environment or Kolla configuration at the paths below. This guide selects Kolla-Ansible `stable/2026.2` and Ubuntu 24.04; do not mix release inventories or environments.

Check the target and destinations first:

```bash
hostname
. /etc/os-release && printf '%s\n' "$PRETTY_NAME"
for p in "$HOME/.venvs/kolla-2026.2" /etc/kolla; do
  if [ -e "$p" ] || [ -L "$p" ]; then printf 'EXISTS: %s — stop and inspect\n' "$p"; else printf 'Absent: %s\n' "$p"; fi
done
```

Continue only on the fresh manager when **both** paths are absent. Never chown or overwrite an existing `/etc/kolla`, which may contain live-cloud secrets. Install prerequisites and create the environment:

```bash
sudo apt update && sudo apt install -y git curl python3-dev python3-venv libffi-dev gcc libssl-dev libdbus-glib-1-dev
```

```bash
mkdir -p "$HOME/.venvs"
/usr/bin/python3 -m venv "$HOME/.venvs/kolla-2026.2"
source "$HOME/.venvs/kolla-2026.2/bin/activate"
python -m pip install --upgrade pip
python -m pip install 'ansible>=13,<15'
python -m pip install 'git+https://opendev.org/openstack/kolla-ansible@stable/2026.2'
```

Run each command only if the preceding command succeeded. If installation fails, stop and investigate rather than continuing with a partial environment. Verify that both executables are in this environment:

```bash
command -v ansible-playbook
command -v kolla-ansible
ansible --version
kolla-ansible --version
```

The executable paths must begin with `$HOME/.venvs/kolla-2026.2/bin/`. Kolla invokes Ansible as a subprocess, so activating the environment matters. In a new terminal, activate it again with `source "$HOME/.venvs/kolla-2026.2/bin/activate"`.

Only after checking `/etc/kolla` is **still absent**, create a private configuration directory and copy the installed release's examples, including its multinode inventory:

```bash
sudo mkdir -m 700 /etc/kolla && sudo chown "$(id -un)":"$(id -gn)" /etc/kolla
```

If creation fails, stop. Then:

```bash
cp -r "$VIRTUAL_ENV/share/kolla-ansible/etc_examples/kolla/." /etc/kolla/ &&
cp -n "$VIRTUAL_ENV/share/kolla-ansible/ansible/inventory/multinode" /etc/kolla/multinode &&
chmod 600 /etc/kolla/passwords.yml &&
kolla-ansible install-deps
```

`install-deps` downloads Ansible collections to the manager; it does not deploy the cloud. Check paths without displaying password contents:

```bash
test -f /etc/kolla/globals.yml && test -f /etc/kolla/passwords.yml && test -f /etc/kolla/multinode && printf '%s\n' 'Example files present'
```

The sample inventory still contains upstream example hosts; **do not target it** until the later inventory step replaces all active sample hosts and verifies group membership. Do not generate or print secrets here.

**Expected result:** Virtualenv executables work, dependency installation succeeds and all three example files exist. **Stop if:** destination already exists, installation or dependency download fails, versions/paths are wrong, or a file is missing. Do not deploy or bootstrap.

## Step 4 — Plan and configure node networking

**Goal:** Give all nodes stable, consistently named management and provider interfaces. **Where:** One physical node console at a time, plus a separate manager terminal for SSH tests. **Prerequisites:** Steps 1–2 complete, correct switch/VLAN and addresses checked, working physical or out-of-band console and a second test shell. A remote SSH connection alone is not a recovery console.

### Inspect before editing

On the console of **node1 only**, then repeat the entire procedure later on node2 and node3:

```bash
hostname
ip -brief link
ip -brief address
ip route
resolvectl status
for p in /sys/class/net/*/address; do printf '%s ' "${p%/address}"; cat "$p"; done
sudo ls -la /etc/netplan
sudo cat /etc/netplan/*.yaml
```

Identify both physical MACs by matching the actual cable/port to this host, not another node's labels. Record existing addresses, routes, DNS resolver IPs and DNS search domains separately. Ensure the gateway and resolver are correct for this LAN and preserve any required routes or DNS search domains in the draft. Review **every** active YAML file for overlapping definitions, including cloud-init-generated files; review cloud-init's network ownership as well. If multiple files, unusual settings, VLANs, bonds or cloud-init regeneration make a safe replacement unclear, stop and design a host-specific migration rather than applying the simple recipe below.

### Back up and draft

The following example assumes exactly one active file, `/etc/netplan/50-cloud-init.yaml`, and no active competing network definitions; if not, stop and adapt all paths and the rollback plan before editing. Confirm the `.disabled`, `.failed`, new file and backup destinations are absent. On this node:

```bash
for p in /etc/netplan/50-cloud-init.yaml.disabled /etc/netplan/90-openstack-lab.yaml /etc/netplan/90-openstack-lab.yaml.failed /root/netplan-pre-openstack.yaml; do
  if sudo test -e "$p"; then printf 'EXISTS: %s — stop and inspect\n' "$p"; else printf 'Absent: %s\n' "$p"; fi
done
```

Only if all are absent, back up and compare the original:

```bash
sudo cp -a /etc/netplan/50-cloud-init.yaml /root/netplan-pre-openstack.yaml &&
sudo cmp -s /etc/netplan/50-cloud-init.yaml /root/netplan-pre-openstack.yaml &&
printf '%s\n' 'Backup verified'
```

If the message does not appear, stop. Create an inactive draft in your home directory; inspect any existing draft first rather than blindly replacing it:

```bash
nano ~/90-openstack-lab.yaml.draft
```

For this simple single-file case, paste the following, replacing **both** MACs with this node's verified MACs, the address with the appropriate one from the table, and the resolver with the inspected DNS server. Preserve required other settings and routes. `192.168.0.1` is the planned gateway, not automatically the DNS server.

```yaml
network:
  version: 2
  ethernets:
    mgmt0:
      match:
        macaddress: "THIS_NODE_MANAGEMENT_MAC"
      set-name: mgmt0
      dhcp4: false
      dhcp6: false
      addresses: ["THIS_NODE_ADDRESS/24"]
      routes:
        - to: default
          via: "192.168.0.1"
      nameservers:
        addresses: ["VERIFIED_DNS_SERVER_IP"]
    provider0:
      match:
        macaddress: "THIS_NODE_PROVIDER_MAC"
      set-name: provider0
      dhcp4: false
      dhcp6: false
      optional: true
```

Save with Ctrl+O, Enter, Ctrl+X, then review `cat ~/90-openstack-lab.yaml.draft`. Check indentation, both MACs, address, DNS and routes. The provider NIC must have no configured IPv4 address or default route; an automatic IPv6 link-local address is not a provider-subnet address.

### Stage, try and verify

Only after all checks pass and the recovery console is ready, on that node install the reviewed draft and park the old file. `install` here copies a file; it does not install software. The commands stop at the first error:

```bash
sudo test ! -e /etc/netplan/90-openstack-lab.yaml &&
sudo test ! -L /etc/netplan/90-openstack-lab.yaml &&
sudo install -o root -g root -m 600 ~/90-openstack-lab.yaml.draft /etc/netplan/90-openstack-lab.yaml &&
sudo mv -n /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.disabled &&
! sudo test -e /etc/netplan/50-cloud-init.yaml &&
sudo test -f /etc/netplan/50-cloud-init.yaml.disabled &&
sudo netplan generate
```

The checks after `mv -n` matter: on some systems it exits successfully even when it skipped an existing destination. If the old filename remains, stop; do not treat the move as successful.

If anything fails, stop; inspect which files actually exist. Do not reboot while the file set is unverified. If both the new file and the parked old file exist and the original filename is absent, restore this specific file set as follows (first ensure the `.failed` destination is absent):

```bash
sudo mv -n /etc/netplan/90-openstack-lab.yaml /etc/netplan/90-openstack-lab.yaml.failed &&
! sudo test -e /etc/netplan/90-openstack-lab.yaml &&
sudo test -f /etc/netplan/90-openstack-lab.yaml.failed &&
sudo mv -n /etc/netplan/50-cloud-init.yaml.disabled /etc/netplan/50-cloud-init.yaml &&
! sudo test -e /etc/netplan/50-cloud-init.yaml.disabled &&
sudo test -f /etc/netplan/50-cloud-init.yaml &&
sudo netplan generate
```

Check the filenames and generated result; if a partial staging failure left a different file set, stop and restore manually from the verified backup rather than running this blindly. `netplan generate` validates/generates but does not apply the network.

From the **same node's recovery console**, try the new network:

```bash
sudo netplan try
```

While its confirmation countdown runs, check from a second node console/TTY:

```bash
ip -brief link
ip -brief address
ip route
resolvectl status
ping -c 3 192.168.0.1
```

Also from the **manager**, substitute the administrator username and this node's new address:

```bash
ssh -i "$HOME/.ssh/openstack_lab_ed25519" ADMIN_USER@THIS_NODE_ADDRESS hostname
```

Accept at the `netplan try` prompt only if SSH returns the right hostname, `mgmt0` has the planned address and default route via `192.168.0.1`, DNS is correct, the gateway responds, and `provider0` shows `LOWER_UP` without a configured provider address or default route. If any check fails, do not accept: let the timer expire, verify runtime rollback from the console, and restore the on-disk file set as above before reboot or retry. Netplan timeout does **not** guarantee the filenames themselves are restored. Verify actual runtime and files, and keep the backup.

Repeat inspection, backup, drafting, try and verification separately for node2 and node3, using each host's own MACs and address. Do not assume an SSH host-key warning after an address change is safe: independently verify its fingerprint.

### Hostname resolution

On **each of the four systems**, inspect and back up `/etc/hosts` before editing:

```bash
cat /etc/hosts
if sudo test -e /root/hosts-pre-openstack; then
  printf '%s\n' 'Backup exists; stop and inspect.'
else
  sudo cp -a /etc/hosts /root/hosts-pre-openstack && sudo cmp -s /etc/hosts /root/hosts-pre-openstack && printf '%s\n' 'Backup verified'
fi
```

Only after a verified backup, edit with `sudo nano /etc/hosts`. Preserve localhost and IPv6 lines; add exactly one mapping per node: `192.168.0.200 node1`, `192.168.0.201 node2`, `192.168.0.202 node3`. Replace a node's existing `127.0.1.1 nodeX` mapping instead of adding a conflicting duplicate. Keep the manager's own local mapping; add a manager LAN alias on other hosts only if its IP is static or DHCP-reserved. Then run on **each** system:

```bash
getent hosts node1
getent hosts node2
getent hosts node3
```

Each should return its planned address. Resolve duplicates or DNS/NSS precedence if not. Verify SSH to all three final addresses again from the manager; inventory-based Ansible tests follow after the release inventory is configured.

**Expected result:** All three nodes have working management IP, DNS, gateway and SSH; each has a distinct, linked `provider0` on the same intended home LAN with no host provider IP; hostnames resolve consistently. **Stop if:** any network check, rollback, resolution or SSH test fails, or if the switch/VLAN and address allocation are unverified. Proceed to inventory configuration only after all three pass. This is connectivity preparation, not proof that floating IPs or OpenStack work.
