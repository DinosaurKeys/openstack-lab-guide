# Foundations: Steps 1–4

> **Private draft: fresh install only.** These instructions describe a new OpenStack installation. Joe reports that the three physical nodes have already been reinstalled, while R2D2 retains files from the previous lab. Their present state needs to be verified; do not assume the previous cloud is running or that its data is gone. Nothing in this guide alone authorizes a rebuild or disk initialization.

**How to use this chapter.** Each step is a sequence of numbered gates. Run one gate, check its expected result, then continue. A gate is not complete until its own verification passes, and passing one gate never completes the next.

Kolla-Ansible runs on a management VM and configures the physical nodes through SSH and Ansible. The VM is an automation workstation, not a fourth OpenStack node. Each of the three physical nodes will run control, network and compute roles. Having three control nodes does not automatically make every service or storage backend highly available.

| System | Planned management address | Purpose |
|---|---|---|
| `manager` VM | Unique, reachable address | Deployment workstation; it may use a routed VMware NAT network if it can reach the nodes. |
| `node1` | `192.168.0.200/24` | control, network, compute |
| `node2` | `192.168.0.201/24` | control, network, compute |
| `node3` | `192.168.0.202/24` | control, network, compute |

The home LAN is `192.168.0.0/24`, with gateway `192.168.0.1`. The planned shared API VIP is `192.168.0.150`; the planned Neutron external allocation pool is `192.168.0.151–192.168.0.199`. These ranges are reserved outside DHCP. Before assigning **any** address, check current router leases, static allocations and network-owner records for conflicts. An unanswered ping is not proof an address is unused. The VIP and floating IPs are not assigned to a host in these steps.

Connect each node's management and provider cables to the same home switch and the intended untagged LAN/VLAN. Use **separate physical ports**, not the same NIC under two names. `mgmt0` has the host management address and default route; `provider0` is a Layer-2-only uplink for a later Neutron external bridge. As with separate VMware uplinks, cable and VLAN placement matter. A link light does not prove that upstream routing or firewall access works. Record the actual MAC and cable for each role on each node. Do not bridge the ports manually. Later Neutron provider-network configuration must use this LAN, gateway and reserved pool consistently.

References: [Kolla 2026.2 quickstart](https://docs.openstack.org/kolla-ansible/2026.2/user/quickstart.html), [multinode deployment](https://docs.openstack.org/kolla-ansible/2026.2/user/multinode.html), [support matrix](https://docs.openstack.org/kolla-ansible/2026.2/user/support-matrix), [Neutron networking](https://docs.openstack.org/kolla-ansible/2026.2/reference/networking/neutron.html), [Ubuntu Server install](https://ubuntu.com/tutorials/install-ubuntu-server), and [Netplan reference](https://netplan.readthedocs.io/en/stable/netplan-yaml/).

## Step 1: Install and identify the systems

**Goal:** Prepare four Ubuntu Server 24.04 systems without touching the existing SATA LVM pools. **Where:** Use the VM console for `manager` and a physical or out-of-band console for each node. **Prerequisites:** An independently verified backup appropriate to anything at risk, installation media, console access, a confirmed NIC and cable layout, and an address-conflict check. These actions are for replacement or fresh systems only.

**Already installed?** Joe's management VM already exists, and he reports that the three nodes have been reinstalled. Do not install Ubuntu again merely to follow this guide. Start with the identity, storage and network checks in item 3. Review any later updates or reboots separately before running them.

1. Install Ubuntu Server 24.04 LTS on the management VM. In the installer on each physical node, positively identify its **NVMe OS disk** by model, serial and capacity. Select only that disk. The existing SATA disks contain LVM pools. Preserve the disks, their partitions and their metadata. Never select them for formatting or automated storage layout. If the installer cannot clearly distinguish devices or proposes changing SATA disks, stop. Installing on the selected NVMe erases its selected contents.
2. Set the hostnames to `manager`, `node1`, `node2` and `node3`. Use the same normal sudo-capable administrator username on each system. Do not enable root SSH login. Give the manager a unique address with working routed access to the nodes. It does not have to be on the same subnet. Nodes may initially use working temporary addresses until Step 4. Install only the base system. Kolla manages OpenStack dependencies later.
3. On **each system's own console**, check the identity and storage before proceeding:

```bash
hostnamectl
. /etc/os-release && printf '%s %s\n' "$PRETTY_NAME" "$VERSION_ID"
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
ip -brief address
ip route
```

On each node, also inspect the LVM metadata without modifying it:

```bash
sudo pvs
sudo vgs
sudo lvs -a
```

Compare disk serials and LVM results with your records. This section does not include `pvcreate`, formatting or Cinder provisioning. Do not assume a disk is unused because it is not mounted.

4. On **each fresh system**, update packages and reboot one system at a time:

```bash
sudo apt update && sudo apt full-upgrade -y
```

If that completes successfully, run `sudo reboot`, reconnect by console, and verify the hostname and address again. These commands alter the operating system and can restart services. This draft does not authorize using them on the live cloud.

5. On each **newly installed system**, set the chosen local timezone and enable network time synchronization. These commands change host settings:

```bash
sudo timedatectl set-timezone Europe/Bucharest
sudo timedatectl set-ntp true
```

Then inspect the result:

```bash
timedatectl show -p Timezone -p NTPSynchronized
```

**Expected:** `Timezone=Europe/Bucharest` and eventually `NTPSynchronized=yes`. Synchronization can take time and requires a working network and a supported time service. If it stays `no`, inspect the active time service, network and upstream time source before moving on. Do not install several time daemons. Step 6 repeats the check across all four machines. Timezone controls local display, not the instant kept by the clock.

**Expected result:** All four individually identified Ubuntu 24.04 systems boot. The SATA pools are still present and untouched, and console and network access work. **Stop if:** disk identities or pool status are uncertain, installation offers to alter a SATA disk, any address conflicts, or a system fails to return after reboot. Resolve these issues before Step 2.

## Step 2: Establish SSH access

**Goal:** Make sure the manager can securely log in to all nodes. **Where:** Start at each system's console, then use the manager terminal. **Prerequisites:** Step 1 complete, working temporary or final management addresses, a sudo-capable account and verified host identities.

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

Stop if any command fails. A fresh manager may not have an `.ssh` directory. Check first and create it only if it is absent. Do not replace an existing directory or follow a symlink:

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

If the owner or permissions are unexpected, stop and inspect the existing directory rather than changing it blindly. Next, check that the dedicated key path does not already exist. Never overwrite an existing key:

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

On the **manager**, substitute the actual username and current address, and repeat once for each node. Verify the fingerprint shown during connection against that node's console before accepting it. Enter initial account passwords only at the local SSH prompt, never in commands or chat:

```bash
ssh-copy-id -i "$HOME/.ssh/openstack_lab_ed25519.pub" ADMIN_USER@NODE_CURRENT_IP
```

Then test each node, replacing both placeholders each time:

```bash
ssh -i "$HOME/.ssh/openstack_lab_ed25519" ADMIN_USER@NODE_CURRENT_IP hostname
```

It must return the expected node hostname without a node account-password prompt. A key passphrase prompt is different. For multiple connections, you can load the key into a local agent with `eval "$(ssh-agent -s)"` then `ssh-add "$HOME/.ssh/openstack_lab_ed25519"`. SSH access does not prove sudo/become access. Test that separately before privileged deployment. Do not disable host-key verification or share credentials.

**Expected result:** All three identity-checked SSH tests return their respective hostnames. **Stop if:** fingerprints differ, authentication fails, or the wrong host answers. Fix that node before proceeding with node-targeting operations. Manager-only installation can still proceed.

### Step 2a: Configure non-interactive sudo for Ansible

**Goal:** Allow the approved `joe` account to run privileged Ansible and Kolla actions without an interactive sudo password prompt. **Where:** Run the following procedure separately on R2D2 and each node. **Prerequisites:** SSH access is established, the target host is verified, and Joe has explicitly accepted the security scope.

This grants `joe` unrestricted root access without a password. It is convenient for unattended Ansible/Kolla operation, but any process running as `joe` can become root. A narrower sudo policy is safer but may block package, networking, storage and Kolla operations.

First inspect the existing rule. Never overwrite it blindly:

```bash
sudo stat /etc/sudoers.d/90-joe-friday 2>/dev/null || \\
  printf '%s\\n' 'File does not exist; it may be created.'
```

If the file exists, inspect and validate it:

```bash
sudo visudo -c
sudo cat /etc/sudoers.d/90-joe-friday
```

If it does not exist, create it with root ownership and mode `0440`:

```bash
if sudo test -e /etc/sudoers.d/90-joe-friday; then
    printf '%s\\n' 'STOP: file already exists; inspect it first.'
else
    printf '%s\\n' 'joe ALL=(ALL:ALL) NOPASSWD: ALL' |
        sudo tee /etc/sudoers.d/90-joe-friday >/dev/null &&
    sudo chown root:root /etc/sudoers.d/90-joe-friday &&
    sudo chmod 0440 /etc/sudoers.d/90-joe-friday &&
    sudo visudo -cf /etc/sudoers.d/90-joe-friday
fi
```

Verify the effective rule without prompting:

```bash
sudo -n id -u
sudo -n whoami
```

Expected output is `0` followed by `root`. If `visudo` reports an error, or either verification fails, stop on that host before continuing. Rollback is removal of only the newly created `/etc/sudoers.d/90-joe-friday` file after confirming no pre-existing file was replaced.

**Expected result:** `sudo -n id -u` returns `0` on R2D2 and all three nodes. This verifies passwordless sudo only; it does not yet verify SSH key authentication or Ansible connectivity.

## Step 3: Install Kolla-Ansible on the manager

**Goal:** Install the release-matched Ansible and Kolla-Ansible tooling on R2D2 without deploying to the nodes. **Where:** R2D2 only. **Prerequisites:** Steps 1–2 complete, working internet/APT access, verified passwordless sudo, and the fresh paths from Step 00.

This build uses the existing fresh virtual environment and configuration directory:

```text
VENV=$HOME/.venvs/kolla-fresh
CONFIG=$HOME/openstack-fresh/kolla
INVENTORY=$HOME/openstack-fresh/kolla/multinode
```

Do not use the historical `/etc/kolla`, `~/.venvs/kolla-2026.2`, or `~/fishbone` paths. The Kolla configuration path is selected through the exported `KOLLA_CONFIG_PATH` variable; the default historical configuration remains untouched.

### Gate 1 of 6 — Confirm R2D2 and activate the fresh environment

Run this on R2D2. It activates the fresh environment and verifies the paths in one pass:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"

hostname
. /etc/os-release && printf '%s\n' "$PRETTY_NAME"
printf 'VENV=%s\nCONFIG=%s\nINVENTORY=%s\n' \
  "$VIRTUAL_ENV" "$KOLLA_CONFIG_PATH" "$KOLLA_INVENTORY"

for p in "$KOLLA_VENV" "$KOLLA_CONFIG_PATH"; do
  if [ -e "$p" ] || [ -L "$p" ]; then
    stat -c 'EXISTS %n owner=%U mode=%a' "$p"
  else
    printf 'MISSING: %s\n' "$p"
  fi
done
```

**Pass when:** the host is R2D2, the prompt shows the fresh environment, both paths exist as Joe-owned directories, and the reported paths are:

```text
VENV=/home/joe/.venvs/kolla-fresh
CONFIG=/home/joe/openstack-fresh/kolla
INVENTORY=/home/joe/openstack-fresh/kolla/multinode
```

**Stop if:** a path is missing, is a symlink, has an unexpected owner or mode, or the prompt does not show the fresh environment.

### Gate 2 of 6 — Install the manager prerequisites

Run on R2D2. This changes R2D2 only:

```bash
sudo apt update &&
sudo apt install -y git curl python3-dev python3-venv libffi-dev gcc libssl-dev libdbus-glib-1-dev
```

**Pass when:** the command finishes without error. No node or OpenStack service is touched.

### Gate 3 of 6 — Install Ansible and Kolla-Ansible

Run on R2D2:

```bash
source "$KOLLA_VENV/bin/activate"
python -m pip install --upgrade pip
python -m pip install 'ansible>=13,<15'
python -m pip install 'git+https://opendev.org/openstack/kolla-ansible@stable/2026.2'
```

Then check where the executables resolve:

```bash
command -v ansible-playbook
command -v kolla-ansible
ansible --version
kolla-ansible --version
```

**Pass when:** both executable paths begin with `$KOLLA_VENV/bin/`, Ansible reports a version inside the release's supported range, and `kolla-ansible --version` succeeds.

**Stop if:** a path resolves outside the fresh environment. Activating the environment matters because Kolla starts Ansible as a subprocess.

### Gate 4 of 6 — Inspect the fresh configuration directory

Read-only check on R2D2:

```bash
find "$KOLLA_CONFIG_PATH" -maxdepth 1 -mindepth 1 -printf '%f\n' | sort
```

**Pass when:** there is no output. An empty directory is safe for the next gate.

**Stop if:** any file is listed. Inspect it instead of overwriting an existing fresh configuration.

### Gate 5 of 6 — Copy the release examples and install collections

Run on R2D2, only after Gate 4 passes:

```bash
cp -r "$VIRTUAL_ENV/share/kolla-ansible/etc_examples/kolla/." "$KOLLA_CONFIG_PATH/" &&
cp -n "$VIRTUAL_ENV/share/kolla-ansible/ansible/inventory/multinode" "$KOLLA_INVENTORY" &&
chmod 600 "$KOLLA_CONFIG_PATH/passwords.yml"
```

This creates the fresh Kolla configuration, the multinode inventory template and the password-file template. It does not contact or change the nodes.

**Pass when:** the command finishes without error.

**Stop if:** any copy fails. A partial configuration is not a usable one.

Then install the Ansible collections:

```bash
kolla-ansible install-deps
```

This installs manager-side dependencies only. It does not bootstrap or deploy the nodes.

**Pass when:** the command finishes without error.

### Gate 6 of 6 — Verify the toolchain and configuration

Read-only check on R2D2:

```bash
test -f "$KOLLA_CONFIG_PATH/globals.yml" &&
test -f "$KOLLA_CONFIG_PATH/passwords.yml" &&
test -f "$KOLLA_INVENTORY" &&
stat -c '%a %n' "$KOLLA_CONFIG_PATH/passwords.yml"
```

**Pass when:** all three files exist and the password file reports mode `600`. Step 3 is complete only at this point.

Do not generate passwords or edit the inventory yet. The copied inventory still contains upstream sample hosts and is not safe to target.

**Step 3 result:** R2D2 has release-matched executables in `~/.venvs/kolla-fresh`, release examples in `~/openstack-fresh/kolla`, and no node-side changes.

## Step 4: Plan and configure node networking

**Goal:** Give all nodes stable, consistently named management and provider interfaces. **Where:** Work on one physical node console at a time, with a separate manager terminal for SSH tests. **Prerequisites:** Steps 1–2 complete, correct switch/VLAN and addresses checked, working physical or out-of-band console and a second test shell. A remote SSH connection alone is not a recovery console.

### Gate 1 of 4 — Inspect before editing

Start on the console of **node1 only**. Repeat the entire procedure later on node2 and node3:

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

Identify both physical MACs by matching the actual cables and ports on this host, not another node's labels. Record existing addresses, routes, DNS resolver IPs and DNS search domains separately. Ensure the gateway and resolver are correct for this LAN and preserve any required routes or DNS search domains in the draft. Review **every** active YAML file, including cloud-init-generated files, for overlapping definitions. Also review cloud-init's network ownership. If multiple files, unusual settings, VLANs, bonds or cloud-init regeneration make a safe replacement unclear, stop and design a host-specific migration rather than applying the simple recipe below.

### Gate 2 of 4 — Back up and draft

The following example assumes exactly one active file, `/etc/netplan/50-cloud-init.yaml`, with no competing active network definitions. If that is not the case, stop and adapt all paths and the rollback plan before editing. Confirm the `.disabled`, `.failed`, new file and backup destinations are absent. On this node:

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

If the message does not appear, stop. Create an inactive draft in your home directory. Inspect any existing draft first rather than blindly replacing it:

```bash
nano ~/90-openstack-lab.yaml.draft
```

For this simple single-file case, paste the following. Replace **both** MACs with this node's verified MACs, the address with the appropriate one from the table, and the resolver with the inspected DNS server. Preserve required other settings and routes. `192.168.0.1` is the planned gateway, not automatically the DNS server.

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

Save with Ctrl+O, Enter, Ctrl+X, then review `cat ~/90-openstack-lab.yaml.draft`. Check indentation, both MACs, address, DNS and routes. The provider NIC must have no configured IPv4 address or default route. An automatic IPv6 link-local address is not a provider-subnet address.

### Gate 3 of 4 — Stage, try and verify

After all checks pass and the recovery console is ready, install the reviewed draft on that node and park the old file. `install` here copies a file; it does not install software. The commands stop at the first error:

```bash
sudo test ! -e /etc/netplan/90-openstack-lab.yaml &&
sudo test ! -L /etc/netplan/90-openstack-lab.yaml &&
sudo install -o root -g root -m 600 ~/90-openstack-lab.yaml.draft /etc/netplan/90-openstack-lab.yaml &&
sudo mv -n /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.disabled &&
! sudo test -e /etc/netplan/50-cloud-init.yaml &&
sudo test -f /etc/netplan/50-cloud-init.yaml.disabled &&
sudo netplan generate
```

The checks after `mv -n` matter: on some systems it exits successfully even when it skipped an existing destination. If the old filename remains, stop. Do not treat the move as successful.

If anything fails, stop and inspect which files actually exist. Do not reboot while the file set is unverified. If both the new file and the parked old file exist and the original filename is absent, restore this specific file set as follows (first ensure the `.failed` destination is absent):

```bash
sudo mv -n /etc/netplan/90-openstack-lab.yaml /etc/netplan/90-openstack-lab.yaml.failed &&
! sudo test -e /etc/netplan/90-openstack-lab.yaml &&
sudo test -f /etc/netplan/90-openstack-lab.yaml.failed &&
sudo mv -n /etc/netplan/50-cloud-init.yaml.disabled /etc/netplan/50-cloud-init.yaml &&
! sudo test -e /etc/netplan/50-cloud-init.yaml.disabled &&
sudo test -f /etc/netplan/50-cloud-init.yaml &&
sudo netplan generate
```

Check the filenames and generated result. If a partial staging failure left a different file set, stop and restore manually from the verified backup rather than running this blindly. `netplan generate` validates and generates the configuration, but does not apply the network.

From the **same node's recovery console**, try the new network:

```bash
sudo netplan try
```

While the confirmation countdown runs, check from a second console/TTY on the node:

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

Accept at the `netplan try` prompt only if SSH returns the right hostname, `mgmt0` has the planned address and default route via `192.168.0.1`, DNS is correct, the gateway responds, and `provider0` shows `LOWER_UP` without a configured provider address or default route. If any check fails, do not accept. Let the timer expire, verify runtime rollback from the console, and restore the on-disk file set as above before rebooting or retrying. A Netplan timeout does **not** guarantee that the filenames themselves are restored. Verify actual runtime and files, and keep the backup.

Repeat the inspection, backup, drafting, try and verification separately for node2 and node3. Use each host's own MACs and address. Do not assume an SSH host-key warning after an address change is safe: independently verify its fingerprint.

### Gate 4 of 4 — Hostname resolution

On **each of the four systems**, inspect and back up `/etc/hosts` before editing:

```bash
cat /etc/hosts
if sudo test -e /root/hosts-pre-openstack; then
  printf '%s\n' 'Backup exists; stop and inspect.'
else
  sudo cp -a /etc/hosts /root/hosts-pre-openstack && sudo cmp -s /etc/hosts /root/hosts-pre-openstack && printf '%s\n' 'Backup verified'
fi
```

Only after verifying the backup, edit with `sudo nano /etc/hosts`. Preserve localhost and IPv6 lines. Add exactly one mapping per node: `192.168.0.200 node1`, `192.168.0.201 node2`, `192.168.0.202 node3`. Replace a node's existing `127.0.1.1 nodeX` mapping instead of adding a conflicting duplicate. Keep the manager's own local mapping. Add a manager LAN alias on other hosts only if its IP is static or DHCP-reserved. Then run on **each** system:

```bash
getent hosts node1
getent hosts node2
getent hosts node3
```

Each lookup should return the node's planned address. If not, resolve duplicates or DNS/NSS precedence. From the manager, verify SSH to all three final addresses again. Inventory-based Ansible tests follow after the release inventory is configured.

**Expected result:** All three nodes have a working management IP, DNS, gateway and SSH. Each has a distinct, linked `provider0` on the same intended home LAN with no host provider IP. Hostnames resolve consistently. **Stop if:** any network check, rollback, resolution or SSH test fails, or if the switch/VLAN and address allocation are unverified. Proceed to inventory configuration only after all three pass. This is connectivity preparation, not proof that floating IPs or OpenStack work.
