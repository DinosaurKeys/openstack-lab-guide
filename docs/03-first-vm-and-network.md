# Steps 10–11: Build your first VM and add a floating IP

> **What this chapter does:** first create one small VM on a private network, then give that same VM a floating IP. Horizon is optional; the CLI path is the main learning path.

## Start here

Run this chapter on **R2D2** only after Step 9 has deployed and validated the cloud. Heat is not required for anything in this chapter: disabling Heat does **not** block VM creation, Neutron networking, floating IPs, or Horizon.

| Your situation | Start with |
|---|---|
| Fresh cloud and no working OpenStack CLI profile | **Step 10A — Connect the OpenStack CLI** |
| Existing working `clouds.yaml` and OpenStack CLI | **Step 10B — Check the current cloud** |
| Private VM already created | **Step 11A — Check the provider-network plan** |
| You only need the GUI | **Optional: Horizon dashboard** at the end |

> **Private working draft:** The addresses below describe this lab and must be sanitized before publication. Running these steps creates cloud resources; writing this guide did not change the cloud. Run OpenStack commands on the **management VM**, as your normal user, in one terminal. At the start of that terminal, run the Step 00 path contract and set the verified `OS_CLOUD` only after reading the generated cloud names without displaying credentials. Stop at each failed gate; never rerun a create command merely because it is slow. Record IDs and which objects you created in a private local record. Names are not necessarily unique: use IDs after checking them.

> **Terminal safety:** A guide block that detects a `STOP` condition now returns only from its temporary function, then prints that the terminal remains open; it does not run `exit` against your R2D2 shell. IDs set by a successful block remain available to the next block.

**Topology — illustration only; DO NOT TYPE**

![Floating IP and outbound network paths](../diagrams/floating-ip-flow.svg)

The floating IP in this example is a **LAN address**, not a public Internet address. `ACTIVE` means the VM build finished; neither that nor an associated floating IP proves guest login, inbound connectivity, outbound SNAT, or DNS.

## Step 10: Build one private test VM

### Step 10A: Connect the OpenStack CLI

**Purpose:** Kolla-Ansible has already deployed the OpenStack services. We now configure the **OpenStack client** so the `openstack` command can authenticate to that deployed cloud. This does **not** redeploy Kolla-Ansible, change service configuration, or create a VM.

#### Kolla-Ansible configuration versus OpenStack client configuration

These are different files for different jobs:

| Configuration | Used by | Purpose |
|---|---|---|
| `$KOLLA_CONFIG_PATH/globals.yml`, `multinode`, and `passwords.yml` | Kolla-Ansible | Deploy and operate the OpenStack services |
| `$HOME/.config/openstack/clouds.yaml` | OpenStack CLI and SDK | Tell client tools how to authenticate to the deployed cloud |

The client file is similar to a VMware client connection profile. It is not the Kolla-Ansible deployment configuration.

#### Why does `clouds.yaml` exist?

After a successful deployment, this command generates a protected source file:

```text
$KOLLA_CONFIG_PATH/clouds.yaml
```

The source is normally protected because it contains authentication data. For normal use, the file is copied once to the standard user location:

```text
$HOME/.config/openstack/clouds.yaml
```

The flow is:

```text
Kolla-Ansible post-deploy
        │ generates
        ▼
$KOLLA_CONFIG_PATH/clouds.yaml       protected Kolla source
        │ copied once, if absent
        ▼
$HOME/.config/openstack/clouds.yaml  Joe's private CLI profile
        │ selected by
        ▼
OS_CLOUD=kolla-admin                 openstack CLI authentication
```

If the destination already exists, it usually means a previous post-deploy/client-setup attempt already created it. The terminal output alone cannot prove exactly which earlier command created it. Do not overwrite it; inspect its owner and mode, then use it if it is owned by Joe and has mode `600`.

**Where:** the management VM. The deployment and configuration validation in Step 9 must have succeeded. Keep cloud configuration, tokens, private keys, console credentials and unredacted logs private.

**Decision:**

- If `$HOME/.config/openstack/clouds.yaml` exists, is owned by Joe, and is mode `600`: **skip the fresh-install copy section**.
- If it is missing during a genuine first-time setup: run the fresh-install section below once.
- If it is a symlink, has unexpected ownership/permissions, or authentication fails: **stop and investigate**. Do not overwrite or regenerate credentials.

**Existing deployment:** If the OpenStack client and a working `clouds.yaml` are already configured, go directly to **Step 10B**. Do not rerun `post-deploy` or overwrite configuration just to open a new terminal.

**STOP for an existing deployment:** Do not continue into the `First-time-only` subsection below. That subsection is only for a genuinely missing client file after a fresh deployment.

**Fresh install only:** activate the same Kolla environment used in Step 9:

```bash
source "$KOLLA_VENV/bin/activate"
```

Install the release-constrained OpenStack CLI on the management VM. Stop if installation fails:

```bash
python -m pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2026.2
command -v openstack
openstack --version
```

**Pass when:** `openstack` resolves inside `$KOLLA_VENV/bin/openstack` and prints its version. This installs the client only; it does not authenticate or create cloud resources.

#### First-time-only: generate the CLI cloud configuration

Run this subsection only when the destination client file is genuinely absent. It is not a normal Step 10 command and it is not required for an already configured cloud.

Run this one self-contained block only when the destination client file is genuinely absent. It does not apply to an already configured client:

```bash
guide_block_01() {
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
command -v kolla-ansible >/dev/null || { printf '%s\n' 'STOP: kolla-ansible is not in the fresh environment.'; return 1; }
read -r -p 'Is passwordless sudo verified on all selected hosts? Type YES or NO: ' SUDO_MODE
case "$SUDO_MODE" in
  YES) export ANSIBLE_BECOME_ASK_PASS=false ;;
  NO) export ANSIBLE_BECOME_ASK_PASS=true ;;
  *) printf '%s\n' 'STOP: answer YES or NO.'; return 1 ;;
esac
kolla-ansible post-deploy -i "$KOLLA_INVENTORY" --configdir "$KOLLA_CONFIG_PATH"
}

if ! guide_block_01; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_01
```

Enter any sudo password only at the local prompt when the interactive mode is selected. Stop if post-deploy fails or its recap reports failed or unreachable hosts. The source and destination below contain credentials. Do not display them. On this fresh-install path, inspect both directory paths and create them **only when absent**. Stop if either path is a symlink, is not a directory, has an unexpected owner or permissions, or if any command fails:

```bash
for d in "$HOME/.config" "$HOME/.config/openstack"; do
  if [ -L "$d" ]; then
    printf 'STOP: symlink requires review: %s\n' "$d"
    break
  elif [ -d "$d" ]; then
    stat -c '%n owner=%U mode=%a' "$d"
  elif [ -e "$d" ]; then
    printf 'STOP: not a directory: %s\n' "$d"
    break
  else
    mkdir -m 700 "$d" || break
    stat -c '%n owner=%U mode=%a' "$d"
  fi
done
```

Continue only after manually confirming that both directories are owned by your normal account, the OpenStack directory is private, and no stop message appeared. Check that the source is nonempty and the destination is absent, without printing the source:

```bash
sudo test -s "$KOLLA_CONFIG_PATH/clouds.yaml" &&
[ ! -e "$HOME/.config/openstack/clouds.yaml" ] &&
[ ! -L "$HOME/.config/openstack/clouds.yaml" ] &&
printf '%s\n' 'Source present; destination absent.'
```

Only after that check passes, create the destination with shell **no-clobber** enabled, owner set to your account and mode 600. If a file or symlink appears before creation, the redirection fails rather than overwriting it. This creates a file containing secrets. Never add it to Git:

```bash
sudo env KOLLA_CONFIG_PATH="$KOLLA_CONFIG_PATH" bash -c 'umask 077; set -C; cat -- "$KOLLA_CONFIG_PATH/clouds.yaml" > "$1" && chown "$2:$3" "$1" && chmod 600 "$1"' \
  _ "$HOME/.config/openstack/clouds.yaml" "$(id -un)" "$(id -gn)"
```

If it fails, stop and inspect the destination ownership and file state without displaying its contents. Do not blindly retry. Check only its metadata:

```bash
stat -c '%n owner=%U mode=%a' "$HOME/.config/openstack/clouds.yaml"
```

### Step 10B: Check the current cloud

**Authentication check:** For this lab's current fresh build, use the environment created in Step 00 (`$KOLLA_VENV`) and the cloud entry generated by the same current configuration. The generated user-facing entry is `kolla-admin`. Do not copy or reuse an old `clouds.yaml`; if your fresh file has a different cloud name, stop and inspect the names privately instead of guessing.

```bash
source "$KOLLA_VENV/bin/activate"
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"
export OS_CLOUD=kolla-admin
openstack token issue -f value -c expires
```

**Expected output:** an expiration time, not a token value. Stop if authentication fails; verify that the client file belongs to the current deployment before changing anything. Inspect the existing deployment and possible collisions:

```bash
openstack service list
openstack endpoint list
openstack compute service list
openstack network agent list
openstack network list --long
openstack subnet list --long
openstack router list
openstack server list --all-projects
openstack image list
openstack flavor list
openstack keypair list
openstack security group list
openstack floating ip list
```

**Expected output:** the intended Keystone, Nova, Neutron, Glance and Placement services and enabled endpoints; each expected `nova-compute`, `nova-scheduler` and `nova-conductor` row is `up` and `enabled` on its intended node, as are the expected network agents. Missing or down entries are a stop condition, not a cue to repeat deployment. The endpoint list shows registration; it does not show whether every endpoint was individually tested. Check service health and capacity, and inspect every matching name and its project, ID, settings and dependencies before reuse. Stop on ambiguous names, conflicting configuration or inaccessible project resources. Do not create another resource with the same name. The commands below are the **fresh path**; where a compatible object exists, explicitly inspect it, use its unique ID and skip its create command. Do not modify shared resources without a separate impact review. Verify `10.254.211.0/24` does not overlap any LAN, VPN, routed network or existing subnet before creating it.

**Existing deployment resume:** Check whether `lab-smoke` currently exists. A missing `lab-smoke` is the expected clean-build state; an existing match is a stop condition for the fresh path.

```bash
guide_block_02() {
VM_MATCHES="$(openstack server list --all-projects --name lab-smoke -f value -c ID)" || return 1
if [ -n "$VM_MATCHES" ]; then
  printf '%s\n' 'STOP: lab-smoke already exists; inspect it before choosing the existing-VM or new-VM path.'
  printf '%s\n' "$VM_MATCHES"
  return 1
fi
printf '%s\n' 'No lab-smoke VM exists; the fresh Step 10 path may continue.'
}

if ! guide_block_02; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_02
```

**Pass when:** the list command succeeds and returns no server ID. **Stop if:** the command fails, more than one match is returned, or any `lab-smoke` VM exists. Do not delete an existing VM as part of the normal guide.

The historical VM observation at `10.254.211.204` is retained only as documentation history; it is not current state and must not block a clean rerun.

### Step 10C: Prepare the key, image and flavor

The guest key must exist **before** the first boot. The local key is reusable across a Step 10 rerun; do not delete or regenerate it just because the cloud keypair was removed. Stop only when one file is missing, either path is a symlink, or the fingerprint check fails:

```bash
guide_block_03() {
if [ -L "$HOME/.ssh/lab-smoke" ] || [ -L "$HOME/.ssh/lab-smoke.pub" ]; then
  printf '%s\n' 'STOP: a Step 10 key path is a symlink.'
  return 1
fi
if [ -e "$HOME/.ssh/lab-smoke" ] && [ -e "$HOME/.ssh/lab-smoke.pub" ]; then
  printf '%s\n' 'Existing local Step 10 keypair found; it will be verified and reused.'
  stat -c '%n owner=%U mode=%a' "$HOME/.ssh/lab-smoke" "$HOME/.ssh/lab-smoke.pub"
elif [ ! -e "$HOME/.ssh/lab-smoke" ] && [ ! -e "$HOME/.ssh/lab-smoke.pub" ]; then
  ssh-keygen -t ed25519 -f "$HOME/.ssh/lab-smoke" -C lab-smoke
else
  printf '%s\n' 'STOP: only one local Step 10 key file exists; inspect before continuing.'
  return 1
fi
}

if ! guide_block_03; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_03
```

Verify the local key before creating or reusing the cloud keypair:

```bash
test -s "$HOME/.ssh/lab-smoke" && test -s "$HOME/.ssh/lab-smoke.pub"
chmod 600 "$HOME/.ssh/lab-smoke"
LOCAL_KEYPAIR_FP="$(ssh-keygen -E md5 -lf "$HOME/.ssh/lab-smoke.pub" | awk '{print $2}' | sed 's/^MD5://')"
printf 'Local key fingerprint: %s\n' "$LOCAL_KEYPAIR_FP"
```

Use the same cloud keypair only when its fingerprint matches the local public key. Create it only when the name is absent:

```bash
guide_block_04() {
if KEYPAIR_LOOKUP_OUTPUT="$(openstack keypair show lab-smoke-key 2>&1)"; then
  CLOUD_KEYPAIR_FP="$(openstack keypair show lab-smoke-key -f value -c fingerprint)" || return 1
  if [ "$CLOUD_KEYPAIR_FP" != "$LOCAL_KEYPAIR_FP" ]; then
    printf 'STOP: cloud/local key fingerprint mismatch: %s != %s\n' "$CLOUD_KEYPAIR_FP" "$LOCAL_KEYPAIR_FP"
    return 1
  fi
  printf '%s\n' 'Cloud keypair exists and matches the local public key; reusing it.'
elif printf '%s' "$KEYPAIR_LOOKUP_OUTPUT" | grep -Eqi 'not found|no keypair|could not be found'; then
  openstack keypair create --public-key "$HOME/.ssh/lab-smoke.pub" lab-smoke-key >/dev/null || return 1
  printf '%s\n' 'Created cloud keypair lab-smoke-key.'
else
  printf '%s\n' 'STOP: keypair lookup failed for a reason other than confirmed not-found.'
  return 1
fi
}

if ! guide_block_04; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_04
```

Verify the cloud keypair before creating the VM:

```bash
openstack keypair show lab-smoke-key -c name -c fingerprint
```

Do not regenerate a lost private key and assume it matches.

Download CirrOS from its [official 0.6.3 release directory](https://download.cirros-cloud.net/0.6.3/); this x86-64 image is a disposable test image, **not production**. CirrOS has a publicly known default login and passwordless sudo: limit network exposure and remove the test VM when no longer needed. Its [publisher](https://github.com/cirros-dev/cirros) documents the guest user `cirros`.

```bash
guide_block_05() {
mkdir -m 700 -p "$HOME/images"
cd "$HOME/images"
if [ -L cirros-0.6.3-x86_64-disk.img ] || [ -d cirros-0.6.3-x86_64-disk.img ]; then
  printf '%s\n' 'STOP: image path is a symlink or directory.'
  return 1
fi
if [ ! -f cirros-0.6.3-x86_64-disk.img ]; then
  tmp_image="$(mktemp "$HOME/images/.cirros-image.XXXXXX")" || return 1
  if ! curl --fail --location --proto '=https' --tlsv1.2 \
      --output "$tmp_image" \
      https://download.cirros-cloud.net/0.6.3/cirros-0.6.3-x86_64-disk.img; then
    rm -f -- "$tmp_image"
    return 1
  fi
  if [ -e cirros-0.6.3-x86_64-disk.img ]; then
    rm -f -- "$tmp_image"
    printf '%s\n' 'STOP: image appeared during download; inspect both files.'
    return 1
  fi
  mv -- "$tmp_image" cirros-0.6.3-x86_64-disk.img
fi
stat -c '%n owner=%U mode=%a size=%s' cirros-0.6.3-x86_64-disk.img
}

if ! guide_block_05; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_05
```

If the checksum file is absent, download it to a temporary file and move it into place only after the transfer succeeds:

```bash
guide_block_06() {
cd "$HOME/images"
if [ -L cirros-0.6.3-SHA256SUMS ] || [ -d cirros-0.6.3-SHA256SUMS ]; then
  printf '%s\n' 'STOP: checksum path is a symlink or directory.'
  return 1
fi
if [ ! -f cirros-0.6.3-SHA256SUMS ]; then
  tmp_sums="$(mktemp "$HOME/images/.cirros-sums.XXXXXX")" || return 1
  if ! curl --fail --location --proto '=https' --tlsv1.2 \
      --output "$tmp_sums" \
      https://download.cirros-cloud.net/0.6.3/SHA256SUMS; then
    rm -f -- "$tmp_sums"
    return 1
  fi
  if [ -e cirros-0.6.3-SHA256SUMS ]; then
    rm -f -- "$tmp_sums"
    printf '%s\n' 'STOP: checksum file appeared during download; inspect both files.'
    return 1
  fi
  mv -- "$tmp_sums" cirros-0.6.3-SHA256SUMS
fi
}

if ! guide_block_06; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_06
```

Verify exactly the intended image checksum:

```bash
grep -E '  cirros-0\.6\.3-x86_64-disk\.img$' cirros-0.6.3-SHA256SUMS | sha256sum -c -
```

**Expected output:** the image filename followed by `OK`. Stop if download or checksum verification fails. This checks consistency with the publisher's checksum over HTTPS, not an independent signature.

### Step 10D: Create or reuse the private network

> **Do not run this section yet if Step 10B failed or was skipped.** An error such as `Missing value auth-url required for auth plugin password` means the OpenStack client context is not loaded; return to Step 10B and make `openstack token issue -f value -c expires` succeed before any resource check or create command.

Only use this fresh-resource path after Step 10B inspection confirms that the intended names do not already exist and that `10.254.211.0/24` does not overlap a connected VPN, routed network or existing subnet. If an object already exists, inspect and reuse it by ID only when it is compatible; do not run its create command again.

Run **one resource command at a time**. After each successful create, record the returned ID, set the variable, and verify the object before continuing. Do not paste the entire sequence into one shell block.

Before the first create, run this collision check. The network, subnet, flavor, image and VM names must be absent. A pre-existing `lab-vm-access` group is handled separately in sub-step E and must be compatible before reuse:

```bash
guide_block_07() {
NETWORK_MATCHES="$(openstack network list --name lab-private -f value -c ID)" || return 1
[ -z "$NETWORK_MATCHES" ] || { printf '%s\n' 'STOP: lab-private already exists.'; return 1; }
SUBNET_MATCHES="$(openstack subnet list --name lab-private-subnet -f value -c ID)" || return 1
[ -z "$SUBNET_MATCHES" ] || { printf '%s\n' 'STOP: lab-private-subnet already exists.'; return 1; }
FLAVORS_JSON="$(openstack flavor list -f json)" || return 1
if printf '%s\n' "$FLAVORS_JSON" | python3 -c 'import json,sys; n="lab.tiny"; rows=json.load(sys.stdin); raise SystemExit(0 if any((r.get("Name") or r.get("name"))==n for r in rows) else 1)'; then
  printf '%s\n' 'STOP: lab.tiny already exists.'
  return 1
fi
IMAGE_MATCHES="$(openstack image list --name lab-test-image -f value -c ID)" || return 1
[ -z "$IMAGE_MATCHES" ] || { printf '%s\n' 'STOP: lab-test-image already exists.'; return 1; }
SECURITY_GROUPS_JSON="$(openstack security group list -f json)" || return 1
MATCHING_SECURITY_GROUPS="$(printf '%s\n' "$SECURITY_GROUPS_JSON" | python3 -c 'import json,sys; n="lab-vm-access"; rows=json.load(sys.stdin); print("\n".join(str(r.get("ID") or r.get("id")) for r in rows if (r.get("Name") or r.get("name"))==n))')"
if [ -n "$MATCHING_SECURITY_GROUPS" ]; then
  printf 'NOTE: lab-vm-access exists and will be inspected for compatible reuse: %s\n' "$MATCHING_SECURITY_GROUPS"
else
  printf '%s\n' 'No existing lab-vm-access group; Step 10E will create it.'
fi
VM_MATCHES="$(openstack server list --all-projects --name lab-smoke -f value -c ID)" || return 1
[ -z "$VM_MATCHES" ] || { printf '%s\n' 'STOP: lab-smoke already exists.'; return 1; }
printf '%s\n' 'Step 10 resource creation path is clear; security-group reuse will be checked separately.'
}

if ! guide_block_07; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_07
```

**Fresh path pass when:** all inspection commands succeed, the network/subnet/flavor/image/VM names are absent, and any existing security group is explicitly identified for compatibility review. **If the check stops because the network, subnet, flavor, or image already exists, do not run A–D below.** Use the existing-resource path instead.

#### Existing-resource path: verify and reuse compatible lab resources

Use this path when Step 10D finds the named network, subnet, flavor, or image already exists. It makes no cloud changes; it confirms there is exactly one of each and places their IDs in this terminal for the later security-group and VM steps.

```bash
guide_block_07_reuse() {
mapfile -t PRIVATE_NET_IDS < <(openstack network list --name lab-private -f value -c ID)
mapfile -t PRIVATE_SUBNET_IDS < <(openstack subnet list --name lab-private-subnet -f value -c ID)
FLAVORS_JSON="$(openstack flavor list -f json)" || return 1
FLAVOR_ID="$(printf '%s\n' "$FLAVORS_JSON" | python3 -c 'import json,sys; n="lab.tiny"; rows=json.load(sys.stdin); print("\n".join(str(r.get("ID") or r.get("id")) for r in rows if (r.get("Name") or r.get("name"))==n))')" || return 1
mapfile -t IMAGE_IDS < <(openstack image list --name lab-test-image -f value -c ID)

if [ "${#PRIVATE_NET_IDS[@]}" -ne 1 ] || [ "${#PRIVATE_SUBNET_IDS[@]}" -ne 1 ] || [ "${#IMAGE_IDS[@]}" -ne 1 ]; then
  printf 'STOP: expected one existing private network, subnet and image; got network=%s subnet=%s image=%s.\n' "${#PRIVATE_NET_IDS[@]}" "${#PRIVATE_SUBNET_IDS[@]}" "${#IMAGE_IDS[@]}"
  return 1
fi
if [ "$(printf '%s\n' "$FLAVOR_ID" | sed '/^$/d' | wc -l)" -ne 1 ]; then
  printf '%s\n' 'STOP: expected exactly one existing lab.tiny flavor.'
  return 1
fi

PRIVATE_NET_ID="${PRIVATE_NET_IDS[0]}"
PRIVATE_SUBNET_ID="${PRIVATE_SUBNET_IDS[0]}"
IMAGE_ID="${IMAGE_IDS[0]}"
openstack network show "$PRIVATE_NET_ID" -c id -c name -c status -c router:external -c subnets
openstack subnet show "$PRIVATE_SUBNET_ID" -c id -c name -c network_id -c cidr -c enable_dhcp
openstack flavor show "$FLAVOR_ID" -c id -c name -c vcpus -c ram -c disk
openstack image show "$IMAGE_ID" -c id -c name -c status -c visibility
printf 'Reusing PRIVATE_NET_ID, PRIVATE_SUBNET_ID, FLAVOR_ID and IMAGE_ID in this terminal.\n'
}

if ! guide_block_07_reuse; then
  printf '%s\n' 'STOP: existing-resource reuse is not safe; the terminal is still open.'
fi
unset -f guide_block_07_reuse
```

**Pass when:** exactly one network, subnet, flavor and image are displayed, the network is `ACTIVE` and internal, the subnet is `10.254.211.0/24` with DHCP enabled, and the image is `active`. Continue at **E — Create and verify the dedicated security group**; skip A–D.

**A — Create and verify the private network:**

```bash
guide_block_08() {
PRIVATE_NET_ID="$(openstack network create lab-private -f value -c id)" || return 1
printf 'PRIVATE_NET_ID=%s\n' "$PRIVATE_NET_ID"
openstack network show "$PRIVATE_NET_ID" -c id -c name -c status -c subnets
}

if ! guide_block_08; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_08
```

**Pass when:** the ID, name `lab-private` and `ACTIVE` status are shown. **Stop if:** the network already exists, the command fails, or the ID is ambiguous.

**B — Create and verify the private subnet:**

```bash
guide_block_09() {
PRIVATE_SUBNET_ID="$(openstack subnet create \
  --network "$PRIVATE_NET_ID" \
  --subnet-range 10.254.211.0/24 \
  --dhcp \
  lab-private-subnet \
  -f value -c id)" || return 1
printf 'PRIVATE_SUBNET_ID=%s\n' "$PRIVATE_SUBNET_ID"
openstack subnet show "$PRIVATE_SUBNET_ID" -c id -c name -c network_id -c cidr -c enable_dhcp
}

if ! guide_block_09; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_09
```

**Pass when:** the subnet is attached to `PRIVATE_NET_ID`, uses `10.254.211.0/24`, and DHCP is enabled. **Stop if:** the CIDR overlaps, the network ID differs, or the subnet already exists.

**C — Create and verify the flavor:**

```bash
guide_block_10() {
FLAVOR_ID="$(openstack flavor create lab.tiny --vcpus 1 --ram 512 --disk 1 -f value -c id)" || return 1
printf 'FLAVOR_ID=%s\n' "$FLAVOR_ID"
openstack flavor show "$FLAVOR_ID" -c id -c name -c vcpus -c ram -c disk
}

if ! guide_block_10; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_10
```

**D — Upload and verify the test image:**

```bash
guide_block_11() {
IMAGE_ID="$(openstack image create lab-test-image \
  --file "$HOME/images/cirros-0.6.3-x86_64-disk.img" \
  --disk-format qcow2 \
  --container-format bare \
  --private \
  -f value -c id)" || return 1
printf 'IMAGE_ID=%s\n' "$IMAGE_ID"
openstack image show "$IMAGE_ID" -c id -c name -c status -c visibility
}

if ! guide_block_11; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_11
```

**Pass when:** image status is `active` and visibility is `private`. **Stop if:** the upload fails, status is not `active`, or the image name already belongs to another object.

**E — Create and verify the dedicated security group:**

A security group may already exist from an earlier attempt. Check before creating it and require exactly zero or one match:

```bash
guide_block_12() {
SECURITY_GROUPS_JSON="$(openstack security group list -f json)" || return 1
GROUP_IDS_OUTPUT="$(printf '%s\n' "$SECURITY_GROUPS_JSON" | python3 -c 'import json,sys; n="lab-vm-access"; rows=json.load(sys.stdin); print("\n".join(str(r.get("ID") or r.get("id")) for r in rows if (r.get("Name") or r.get("name"))==n))')" || return 1
GROUP_IDS=()
if [ -n "$GROUP_IDS_OUTPUT" ]; then
  mapfile -t GROUP_IDS <<< "$GROUP_IDS_OUTPUT"
fi
if [ "${#GROUP_IDS[@]}" -gt 1 ]; then
  printf 'STOP: more than one lab-vm-access group matched.\n'
  return 1
elif [ "${#GROUP_IDS[@]}" -eq 1 ]; then
  ACCESS_GROUP_ID="${GROUP_IDS[0]}"
else
  ACCESS_GROUP_ID="$(openstack security group create lab-vm-access -f value -c id)" || return 1
fi
printf 'ACCESS_GROUP_ID=%s\n' "$ACCESS_GROUP_ID"
openstack security group show "$ACCESS_GROUP_ID" -c id -c name -c project_id
}

if ! guide_block_12; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_12
```

Select the actual IPv4 address of the client that will test the VM. Account for NAT; do not type the literal placeholder and do not use `0.0.0.0/0` for convenience. Set the resulting CIDR only after verifying the source address:

```bash
guide_block_13() {
ip -4 route get 192.168.0.150
read -r -p 'Enter the verified trusted-client IPv4 address: ' TRUSTED_CLIENT_IP
if ! python3 -c 'import ipaddress,sys; ipaddress.IPv4Address(sys.argv[1])' "$TRUSTED_CLIENT_IP"; then
  printf '%s\n' 'STOP: enter a valid IPv4 address with octets from 0 to 255.'
  return 1
fi
TRUSTED_CIDR="${TRUSTED_CLIENT_IP}/32"
printf 'Using trusted source CIDR: %s\n' "$TRUSTED_CIDR"
}

if ! guide_block_13; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_13
```

Before creating ingress rules, inspect the current list and reuse an exact existing rule. This avoids duplicate-rule errors when a partially completed run is resumed:

```bash
guide_block_14() {
if [ -z "${ACCESS_GROUP_ID:-}" ]; then
  printf '%s\n' 'STOP: ACCESS_GROUP_ID is empty; run Step 10E Block 12 first.'
  return 1
fi
if [ -z "${TRUSTED_CIDR:-}" ]; then
  printf '%s\n' 'STOP: TRUSTED_CIDR is empty; run Block 13 first using a bare IPv4 address.'
  return 1
fi
rule_id() {
  local protocol="$1" port="$2" rules_json
  rules_json="$(openstack security group rule list "$ACCESS_GROUP_ID" -f json)" || return 1
  printf '%s\n' "$rules_json" |
    python3 -c '
import json, sys
rows = json.load(sys.stdin)
protocol, port, cidr = sys.argv[1:]
for row in rows:
    if (row.get("Direction") or row.get("direction")) != "ingress":
        continue
    if (row.get("IP Protocol") or row.get("ip_protocol")) != protocol:
        continue
    if (row.get("IP Range") or row.get("ip_range")) != cidr:
        continue
    if port:
        value = row.get("Port Range") or row.get("port_range") or ""
        if value not in (port, f"{port}:{port}"):
            continue
    print(row.get("ID") or row.get("id"))
    break
' "$protocol" "$port" "$TRUSTED_CIDR"
}

SSH_RULE_ID="$(rule_id tcp 22)" || return 1
if [ -n "$SSH_RULE_ID" ]; then
  printf 'Reusing existing SSH_RULE_ID=%s\n' "$SSH_RULE_ID"
else
  SSH_RULE_ID="$(openstack security group rule create \
    --ingress --ethertype IPv4 --protocol tcp --dst-port 22 \
    --remote-ip "$TRUSTED_CIDR" "$ACCESS_GROUP_ID" \
    -f value -c id)" || return 1
  printf 'Created SSH_RULE_ID=%s\n' "$SSH_RULE_ID"
fi
openstack security group rule show "$SSH_RULE_ID"

ICMP_RULE_ID="$(rule_id icmp '')" || return 1
if [ -n "$ICMP_RULE_ID" ]; then
  printf 'Reusing existing ICMP_RULE_ID=%s\n' "$ICMP_RULE_ID"
else
  ICMP_RULE_ID="$(openstack security group rule create \
    --ingress --ethertype IPv4 --protocol icmp \
    --remote-ip "$TRUSTED_CIDR" "$ACCESS_GROUP_ID" \
    -f value -c id)" || return 1
  printf 'Created ICMP_RULE_ID=%s\n' "$ICMP_RULE_ID"
fi
openstack security group rule show "$ICMP_RULE_ID"
}

if ! guide_block_14; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_14
```

Verify the complete group before creating the VM:

```bash
openstack security group rule list "$ACCESS_GROUP_ID"
```

**Pass when:** the intended SSH and ICMP ingress rules are present and the default egress rules are understood. **Stop if:** the CIDR is rejected, the group is shared or ambiguous, or a rule already exists and its ownership is unclear.

Do not create the VM until the network, subnet, flavor, image and security-group IDs have all been recorded and individually verified.

### Step 10E: Boot and verify the VM

Choose a name that does not already exist. On a genuinely fresh installation use `lab-smoke`; if the name belongs to the existing protected VM, stop and choose a **different** name only after separate approval for a new VM. Set the chosen name in this terminal:

```bash
VM_NAME='lab-smoke'
```

Recheck `openstack server list --all-projects` for the chosen name before boot. Use the verified IDs recorded in Step 10D rather than names. The fresh create explicitly supplies only the dedicated security group and keypair:

```bash
guide_block_15() {
VM_ID="$(openstack server create "$VM_NAME" \
  --flavor "$FLAVOR_ID" \
  --image "$IMAGE_ID" \
  --network "$PRIVATE_NET_ID" \
  --security-group "$ACCESS_GROUP_ID" \
  --key-name lab-smoke-key \
  --wait \
  -f value -c id)" || return 1
printf 'VM_ID=%s\n' "$VM_ID"
openstack server show "$VM_ID" -c id -c status -c addresses -c key_name
openstack port list --server "$VM_ID" --long
}

if ! guide_block_15; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_15
```

**Pass when:** the create command returns a server ID and the ID lookup succeeds. **Stop if:** the command fails, the server enters `ERROR`, or the server ID is missing. If the command result is uncertain, inspect `openstack server list --all-projects --name "$VM_NAME"` before retrying; do not create another VM by name.

**Expected output:** `ACTIVE`, a private fixed IP, the intended key name, and a VM port on `lab-private`. Record the server ID, port ID and fixed IPv4. Inspect that port and its actual groups. If still building, inspect the same VM later; if `ERROR`, stop and diagnose safely without creating another. The VM is not yet externally reachable. Continue only after the VM, image and port gates pass.

## Step 11: Give the VM a floating IP

> **Prerequisite — stop here unless Step 10E is complete:** you must have one `ACTIVE` test VM and have recorded its `VM_ID`, `VM_PORT`, `FIXED_IP`, and `PRIVATE_SUBNET_ID`; do not create a router, attach a router interface, allocate a floating IP, or add provider-side security rules before that checkpoint.

### Step 11A: Check the provider-network plan

This path creates a flat external provider network, dedicated router and floating IP; it does **not** reconfigure switches, node interfaces or DHCP. Verify the existing Kolla/Neutron mapping `physnet1:br-ex`, the `provider0` uplink, switch/VLAN and upstream path on the actual hosts before proceeding. A link-up indication alone does not prove that the upstream network works. The external LAN is `192.168.0.0/24`, gateway `192.168.0.1`, and Neutron pool `192.168.0.151–192.168.0.199`. Joe has confirmed that pool is excluded from DHCP; the VIP `.150` is excluded and nodes `.200–.202` are outside the pool. Confirm the pool is also clear of **all** static assignments, current allocations and other Neutron networks with the LAN administrator and the relevant DHCP/IPAM records before assignment; an unanswered ping does not establish availability. The router gateway and floating IP each consume a pool address.

Re-run the inspection in **Step 10B — Authenticate and inspect the existing cloud** and also:

```bash
openstack port list --server "$VM_ID" --long
openstack floating ip list
```

Select the intended port from the output; inspect it, record its original security-group IDs and existing floating IP association, and inspect any router or subnet already attached. Stop for conflicting or ambiguous resources. If an external network, subnet or router already exists, inspect type, physical mapping, CIDR, pool, gateway, SNAT, ownership and all dependent ports. Reuse a network by its ID only if it is fully compatible, and skip its create command. A reused/shared router must not have its gateway or interfaces changed without separate impact approval; if the private subnet is already attached, inspect the existing route rather than creating another router. Never allocate or associate a second FIP if one already exists on the target port without explicit review.

Discover the selected VM port and its fixed IP from the verified server. This expects exactly one port and one fixed IP; stop if the VM has multiple ports or addresses and inspect them manually:

```bash
guide_block_16() {
mapfile -t VM_PORT_IDS < <(openstack port list --server "$VM_ID" -f value -c ID)
if [ "${#VM_PORT_IDS[@]}" -ne 1 ]; then
  printf 'STOP: expected exactly one VM port; found %s.\n' "${#VM_PORT_IDS[@]}"
  return 1
fi
VM_PORT="${VM_PORT_IDS[0]}"
PORT_JSON="$(openstack port show "$VM_PORT" -f json)" || return 1
PORT_VALUES="$(printf '%s\n' "$PORT_JSON" | python3 -c 'import json,sys; d=json.load(sys.stdin); f=d.get("fixed_ips", []); raise SystemExit("expected exactly one fixed IP") if len(f) != 1 else print(f[0]["ip_address"], f[0]["subnet_id"])')" || return 1
read -r FIXED_IP PRIVATE_SUBNET_ID <<< "$PORT_VALUES"
printf 'VM_PORT=%s\nFIXED_IP=%s\nPRIVATE_SUBNET_ID=%s\n' "$VM_PORT" "$FIXED_IP" "$PRIVATE_SUBNET_ID"
openstack port show "$VM_PORT"
}

if ! guide_block_16; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_16
```

Choose the resource path before running any create command. Each resource has a separate existing-resource branch and new-resource branch. Never run a create command in the existing-resource branch.

**A — Identify or create the external network:**

```bash
guide_block_17() {
mapfile -t EXT_NET_IDS < <(openstack network list --name lab-external -f value -c ID)
if [ "${#EXT_NET_IDS[@]}" -gt 1 ]; then
  printf '%s\n' 'STOP: more than one lab-external network matched.'
  return 1
elif [ "${#EXT_NET_IDS[@]}" -eq 1 ]; then
  EXT_NET_ID="${EXT_NET_IDS[0]}"
  NETWORK_JSON="$(openstack network show "$EXT_NET_ID" -f json)" || return 1
  printf '%s\n' "$NETWORK_JSON" | python3 -c '
import json, sys
d = json.load(sys.stdin)
expected = {"status": "ACTIVE", "is_router_external": True, "provider_network_type": "flat", "provider_physical_network": "physnet1"}
errors = [f"{k}={d.get(k)!r}, expected {v!r}" for k, v in expected.items() if d.get(k) != v]
if errors:
    print("STOP: existing external network is incompatible: " + "; ".join(errors))
    raise SystemExit(1)
print("Existing compatible external network verified.")
'
else
  EXT_NET_ID="$(openstack network create \
    --external \
    --provider-network-type flat \
    --provider-physical-network physnet1 \
    lab-external \
    -f value -c id)" || return 1
  printf 'EXT_NET_ID=%s\n' "$EXT_NET_ID"
  openstack network show "$EXT_NET_ID"
fi
printf 'EXT_NET_ID=%s\n' "$EXT_NET_ID"
}

if ! guide_block_17; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_17
```

**Pass when:** exactly one `lab-external` network is present and its status is `ACTIVE`, external flag is true, provider type is `flat`, and provider physical network is `physnet1`. **Stop if:** it is missing, duplicated, or incompatible. Do not repair an incompatible existing network inside this learning step; stop and review it separately.

**B — Identify or create the external subnet:**

```bash
guide_block_18() {
mapfile -t EXT_SUBNET_IDS < <(openstack subnet list --name lab-external-subnet -f value -c ID)
if [ "${#EXT_SUBNET_IDS[@]}" -gt 1 ]; then
  printf '%s\n' 'STOP: more than one lab-external-subnet matched.'
  return 1
elif [ "${#EXT_SUBNET_IDS[@]}" -eq 1 ]; then
  EXT_SUBNET_ID="${EXT_SUBNET_IDS[0]}"
  openstack subnet show "$EXT_SUBNET_ID" -c id -c name -c network_id -c cidr -c gateway_ip -c enable_dhcp -c allocation_pools
else
  EXT_SUBNET_ID="$(openstack subnet create \
    --network "$EXT_NET_ID" \
    --subnet-range 192.168.0.0/24 \
    --gateway 192.168.0.1 \
    --allocation-pool start=192.168.0.151,end=192.168.0.199 \
    --no-dhcp \
    lab-external-subnet \
    -f value -c id)" || return 1
  printf 'EXT_SUBNET_ID=%s\n' "$EXT_SUBNET_ID"
  openstack subnet show "$EXT_SUBNET_ID" -c id -c name -c network_id -c cidr -c gateway_ip -c enable_dhcp -c allocation_pools
fi
printf 'EXT_SUBNET_ID=%s\n' "$EXT_SUBNET_ID"
}

if ! guide_block_18; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_18
```

**Pass when:** exactly one subnet is present, attached to `EXT_NET_ID`, uses `192.168.0.0/24`, gateway `192.168.0.1`, allocation pool `192.168.0.151–192.168.0.199`, and DHCP is disabled. **Stop if:** any value differs. Do not modify an incompatible existing subnet here.

**C — Identify or create the router:**

```bash
guide_block_19() {
mapfile -t ROUTER_IDS < <(openstack router list --name lab-router -f value -c ID)
if [ "${#ROUTER_IDS[@]}" -gt 1 ]; then
  printf '%s\n' 'STOP: more than one lab-router matched.'
  return 1
elif [ "${#ROUTER_IDS[@]}" -eq 1 ]; then
  ROUTER_ID="${ROUTER_IDS[0]}"
  openstack router show "$ROUTER_ID"
else
  ROUTER_ID="$(openstack router create lab-router -f value -c id)" || return 1
  printf 'ROUTER_ID=%s\n' "$ROUTER_ID"
  openstack router show "$ROUTER_ID"
fi
printf 'ROUTER_ID=%s\n' "$ROUTER_ID"
}

if ! guide_block_19; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_19
```

**Pass when:** exactly one router is identified and its dependencies are understood. **Stop if:** the router is shared, has an unexpected gateway or interface, or its ownership is unclear. Do not mutate an existing/shared router in this learning step.

**D — Set the gateway on the newly created dedicated router:**

Run this only when the router was created by this exercise:

```bash
openstack router set --external-gateway "$EXT_NET_ID" --enable-snat "$ROUTER_ID"
```

Verify before attaching the private subnet:

```bash
openstack router show "$ROUTER_ID"
```

**Pass when:** the router shows the intended external network gateway and SNAT state. **Stop if:** the router is shared, already has an unexpected gateway, or the mutation fails.

**E — Attach the private subnet:**

Run this only after the gateway verification passes and only when the private subnet is not already attached:

```bash
openstack router add subnet "$ROUTER_ID" "$PRIVATE_SUBNET_ID"
```

Verify the attachment:

```bash
openstack router show "$ROUTER_ID"
openstack port list --router "$ROUTER_ID"
```

**Pass when:** the router has the intended external gateway and exactly the expected private-subnet interface. **Stop if:** the subnet is already attached, the router state differs, or the interface is unexpected.

For an existing/shared router, do not run either mutation block without separate impact approval.

For either path, inspect the selected IDs:

```bash
openstack network show "$EXT_NET_ID"
openstack subnet show "$EXT_SUBNET_ID"
openstack router show "$ROUTER_ID"
openstack port list --router "$ROUTER_ID"
```

**Expected output:** external flat `physnet1` network, matching no-DHCP subnet and allocation pool, router with SNAT enabled, external gateway port on the intended subnet, and an interface on the recorded private subnet. Stop if not. In a multi-subnet external network, explicitly verify gateway subnet selection rather than assuming it.

### Step 11B: Allocate and attach the floating IP

**A — Select and verify the security group:**

For a fresh VM, use the dedicated group created and verified in Step 10D. For an existing VM, inspect its attached group IDs first and reuse a compatible dedicated group; do not recreate a group that already exists.

Enter the verified security-group UUID from the previous inspection; do not type a placeholder:

```bash
guide_block_20() {
read -r -p 'Enter verified security-group UUID: ' ACCESS_GROUP_ID
if [[ ! "$ACCESS_GROUP_ID" =~ ^[0-9a-fA-F-]{36}$ ]]; then
  printf '%s\n' 'STOP: the security-group ID is not a UUID.'
  return 1
fi
openstack security group show "$ACCESS_GROUP_ID" -c id -c name -c project_id
openstack security group rule list "$ACCESS_GROUP_ID"
}

if ! guide_block_20; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_20
```

If no suitable group exists, stop and create a separate dedicated group only after approval. Record its returned ID, then rerun the verification above. Never put the literal placeholder into a command.

**B — Verify or add the approved ingress rules:**

Select the actual IPv4 address of the client that will test the floating IP, accounting for NAT. Do not use the API VIP, the floating-IP pool, or `0.0.0.0/0` as the source. Set this only after verifying the source address:

```bash
guide_block_21() {
ip -4 route get 192.168.0.150
read -r -p 'Enter the verified trusted-client IPv4 address: ' TRUSTED_CLIENT_IP
if ! python3 -c 'import ipaddress,sys; ipaddress.IPv4Address(sys.argv[1])' "$TRUSTED_CLIENT_IP"; then
  printf '%s\n' 'STOP: enter a valid IPv4 address with octets from 0 to 255.'
  return 1
fi
TRUSTED_CIDR="${TRUSTED_CLIENT_IP}/32"
printf 'Using trusted source CIDR: %s\n' "$TRUSTED_CIDR"
}

if ! guide_block_21; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_21
```

Inspect the verified group and reuse an exact existing rule. This block adds only missing rules:

```bash
guide_block_22() {
rule_id() {
  local protocol="$1" port="$2" rules_json
  rules_json="$(openstack security group rule list "$ACCESS_GROUP_ID" -f json)" || return 1
  printf '%s\n' "$rules_json" |
    python3 -c '
import json, sys
rows = json.load(sys.stdin)
protocol, port, cidr = sys.argv[1:]
for row in rows:
    if (row.get("Direction") or row.get("direction")) != "ingress":
        continue
    if (row.get("IP Protocol") or row.get("ip_protocol")) != protocol:
        continue
    if (row.get("IP Range") or row.get("ip_range")) != cidr:
        continue
    if port:
        value = row.get("Port Range") or row.get("port_range") or ""
        if value not in (port, f"{port}:{port}"):
            continue
    print(row.get("ID") or row.get("id"))
    break
' "$protocol" "$port" "$TRUSTED_CIDR"
}

ICMP_RULE_ID="$(rule_id icmp '')" || return 1
if [ -n "$ICMP_RULE_ID" ]; then
  printf 'Reusing existing ICMP_RULE_ID=%s\n' "$ICMP_RULE_ID"
else
  ICMP_RULE_ID="$(openstack security group rule create \
    --ingress --ethertype IPv4 --protocol icmp \
    --remote-ip "$TRUSTED_CIDR" "$ACCESS_GROUP_ID" \
    -f value -c id)" || return 1
  printf 'Created ICMP_RULE_ID=%s\n' "$ICMP_RULE_ID"
fi
openstack security group rule show "$ICMP_RULE_ID"

read -r -p 'Create or reuse the approved SSH rule now? Type YES to continue: ' SSH_APPROVAL
if [ "$SSH_APPROVAL" = 'YES' ]; then
  SSH_RULE_ID="$(rule_id tcp 22)" || return 1
  if [ -n "$SSH_RULE_ID" ]; then
    printf 'Reusing existing SSH_RULE_ID=%s\n' "$SSH_RULE_ID"
  else
    SSH_RULE_ID="$(openstack security group rule create \
      --ingress --ethertype IPv4 --protocol tcp --dst-port 22 \
      --remote-ip "$TRUSTED_CIDR" "$ACCESS_GROUP_ID" \
      -f value -c id)" || return 1
    printf 'Created SSH_RULE_ID=%s\n' "$SSH_RULE_ID"
  fi
  openstack security group rule show "$SSH_RULE_ID"
else
  printf '%s\n' 'SSH rule not changed.'
fi
}

if ! guide_block_22; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_22
```

Verify the complete group and port state:

```bash
openstack security group rule list "$ACCESS_GROUP_ID"
openstack port show "$VM_PORT"
```

**Pass when:** the intended group is attached to the selected port and the required ingress rules are present. **Stop if:** the CIDR is rejected, the group is shared or ambiguous, a rule already exists with unclear ownership, or the port has unexpected security groups.

If the group is **not already attached** to the selected port and this change is approved, add it and record that this exercise added the association:

```bash
openstack port set --security-group "$ACCESS_GROUP_ID" "$VM_PORT"
openstack port show "$VM_PORT"
```

This adds a group without removing existing groups. Compare the port's complete group list with the original. Broad rules in any other group can still allow access. Check port security and guest firewall too. Do not alter a shared security group to make a test pass.

Allocate exactly one floating IP from the verified pool and capture its returned ID:

```bash
guide_block_23() {
FIP_ID="$(openstack floating ip create --subnet "$EXT_SUBNET_ID" "$EXT_NET_ID" -f value -c id)" || return 1
printf 'FIP_ID=%s\n' "$FIP_ID"
}

if ! guide_block_23; then
  printf '%s\n' 'STOP: this check failed; the terminal is still open.'
fi
unset -f guide_block_23
```

Stop here and verify that the returned address belongs to the approved pool and is not already assigned:

```bash
openstack floating ip show "$FIP_ID"
```

Only after the FIP is verified as unassociated, attach it to the recorded VM port and fixed IP:

```bash
openstack floating ip set \
  --port "$VM_PORT" \
  --fixed-ip-address "$FIXED_IP" \
  "$FIP_ID"
```

Verify the association:

```bash
openstack floating ip show "$FIP_ID"
openstack port show "$VM_PORT"
openstack server show "$VM_ID" -c status -c addresses
```

**Expected output:** the FIP shows the recorded VM port, fixed IP and router; the VM remains `ACTIVE`. Record the actual floating address for the tests below. An API association does not test reachability.

### Step 11C: Test SSH, outbound access and DNS

From the trusted LAN client that already holds the matching private key, derive the allocated address from the verified FIP ID. Keep private keys on the machine where they were created; if that client lacks an authorized key, provision a separate public key through an approved method rather than moving a private key.

```bash
FLOATING_IP="$(openstack floating ip show "$FIP_ID" -f value -c floating_ip_address)"
printf 'FLOATING_IP=%s\n' "$FLOATING_IP"
ping -c 3 "$FLOATING_IP"
ssh -i "$HOME/.ssh/lab-smoke" "cirros@$FLOATING_IP"
```

**Expected:** ping replies if ICMP is allowed, then SSH login as `cirros` after verifying the host key by a trusted console. Failure can be upstream filtering, security groups, routing, guest firewall/service, or key injection; a successful ping is not SSH proof. This tests LAN reachability, not public access.

Choose a known reachable external IP and a domain appropriate to your network. Enter real test values; do not type placeholders:

```bash
read -r -p 'Enter a known reachable external IPv4 address: ' EXTERNAL_IP
read -r -p 'Enter a known DNS test domain: ' TEST_DOMAIN
ping -c 3 10.254.211.1
ping -c 3 "$EXTERNAL_IP"
nslookup "$TEST_DOMAIN"
```

The first tests the private gateway (confirm its actual address from subnet/router if not `.1`); the second tests outbound IP connectivity via SNAT and upstream routing, without relying on DNS; the third tests DNS resolution. ICMP may be blocked even with working routes. A failed lookup can be resolver, DNS egress or upstream policy; a successful lookup alone does not prove outbound application access. Note each PASS, FAIL or NOT TESTED independently. Never claim live connectivity from a simulated or API-only check.

### Step 11D: Remove only this exercise's changes

Rollback is optional, and it changes real cloud state. Before each command, inspect exact IDs, current associations and dependencies; stop on unexpected changes or concurrent use. Never touch preexisting resources, other ports, unrelated security groups, or shared routers. Keep a record of what this exercise actually created or attached.

1. If **this exercise** associated the recorded FIP and it still points to `VM_PORT`, disassociate it and inspect the result. Skip this block if it was never associated:

   ```bash
   openstack floating ip show "$FIP_ID"
   openstack floating ip unset --port "$FIP_ID"
   openstack floating ip show "$FIP_ID"
   ```

   Only if **this exercise allocated that exact FIP** and it is now unassociated, release it. Never delete a reused FIP:

   ```bash
   openstack floating ip delete "$FIP_ID"
   ```
2. Only if this exercise added the recorded security group to a preexisting port, and the association is still the same, remove that one group by ID and compare with the recorded originals:

   ```bash
   openstack port unset --security-group "$ACCESS_GROUP_ID" "$VM_PORT"
   openstack port show "$VM_PORT"
   ```

3. Only for a newly created dedicated router, after verifying no other dependencies, remove this exercise's private interface and external gateway, then delete it:

   ```bash
   openstack router remove subnet "$ROUTER_ID" "$PRIVATE_SUBNET_ID"
   openstack router unset --external-gateway "$ROUTER_ID"
   openstack router delete "$ROUTER_ID"
   ```

4. For full exercise cleanup, delete a **newly created test VM only after explicit approval**, verifying its ID and that no data must be preserved. Wait until deletion completes and its port is gone before deleting any newly created security group, private subnet/network, image or flavor. Delete the new external subnet/network only after confirming no router, floating IP, port or other dependency remains. Use exact IDs with `openstack server delete`, `security group delete`, `subnet delete`, `network delete`, `image delete` and `flavor delete` in dependency order. Delete a newly created keypair only when no longer needed; preserve the local private key until the operator has decided it is no longer needed. Never delete the already deployed `lab-smoke` as a rollback of this exercise. Verify each target is gone with its corresponding `show` or list command.

**Completion gate:** record the VM/port and FIP IDs, intended associations, and separate inbound SSH/ICMP, outbound IP and DNS results. Unperformed tests remain **NOT TESTED**; no network mutation or connectivity test has been performed merely by reading this guide.

## Optional: Horizon dashboard

Horizon is optional. It is a web GUI for the same OpenStack services used by the CLI. It is not required to create or verify the first VM and network, and it is not a separate deployment step. Complete the CLI authentication and Step 10–11 gates first. Use either the CLI or Horizon for an individual operation; do not create the same resource twice through both interfaces.

After the deploy and validation gates pass, open the Horizon URL through the configured API VIP:

```text
http://192.168.0.150/
```

The page should redirect to the Horizon login screen. Use the Keystone administrator account:

```text
Username: admin
```

The Horizon administrator password is the generated `keystone_admin_password` value in the protected `$KOLLA_CONFIG_PATH/passwords.yml` file. Retrieve it only on the local management VM using an approved secure method. If no password manager is available, use the read-only editor view below, search for `keystone_admin_password`, enter the value directly into the browser, then exit without saving:

```bash
nano --view "$KOLLA_CONFIG_PATH/passwords.yml"
```

Never run `cat`, `grep`, `print`, or a script that displays the password; never place it in chat, Git, shell history, environment variables, screenshots or command output.

Do **not** run `kolla-genpwd` after the cloud has been deployed just to recover a forgotten Horizon password. That can generate new service secrets and make the deployed cloud inconsistent. If the password must be changed rather than retrieved, stop and use the controlled Kolla password-rotation procedure: back up `passwords.yml`, change only the intended secret, set `update_keystone_service_user_passwords: true` for the controlled reconfigure, run the approved reconfigure, then return the setting to `false`. Password rotation is a live change and requires a separate backup, impact review and approval. See the [Kolla-Ansible 2026.2 password rotation guide](https://docs.openstack.org/kolla-ansible/2026.2/admin/password-rotation.html).

**Pass when:** the Horizon login page responds through the VIP and the administrator can authenticate without exposing the password. A working login does not by itself prove that every OpenStack API, service or VM workflow is healthy.