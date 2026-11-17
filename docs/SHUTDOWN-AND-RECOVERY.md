# Shutdown, Reboot and Recovery

**Purpose:** safely shut down the three-node Kolla-Ansible lab, check it after reboot, recover only when the evidence requires recovery, and restart one approved service when appropriate.

This guide is for the current lab paths:

- Management VM: `r2d2`
- Kolla virtual environment: `$HOME/.venvs/kolla-fresh`
- Kolla configuration: `$HOME/openstack-fresh/kolla`
- Kolla inventory: `$HOME/openstack-fresh/kolla/multinode`
- API VIP: `192.168.0.150`
- Nodes: `node1`, `node2`, `node3`

This guide is adapted to the current fresh-lab paths. The earlier [openstack-from-zero recovery guide](https://github.com/DinosaurKeys/openstack-from-zero/blob/main/docs/reboot-and-recovery.md) describes a different deployment; do not run its scripts unchanged here.

## Quick recovery path

1. Power on the nodes and check SSH and Docker (Section 3A).
2. Check Galera (Section 4B). If it is healthy, **do not recover it**. If all members have lost the Primary component, follow the decision and recovery steps in Sections 4C–4D.
3. Check RabbitMQ membership and each application (Sections 5 and 5A). A stopped application and a repeatedly restarting container require different actions. Do not repeat `start_app` when the container is cycling.
4. Check the API, Placement, and service heartbeats (Section 5B). Do not call the cloud healthy until the database, broker, and required services pass separately.

**Current known limitation:** this guide does not yet provide a verified one-command repair for a RabbitMQ quorum-queue log recovery timeout. Preserve the broker data and investigate that specific failure; do not delete its volume or reset the cluster as a shortcut.

## Safety rules

1. **Health checks come first.** A failed port check, `502`, `starting` container or unavailable API is a symptom, not automatic proof that recovery is required.
2. **A reboot does not automatically require recovery.** Allow the hosts and containers time to start, then verify the API, MariaDB/Galera, RabbitMQ and service heartbeats.
3. **Do not run `kolla-genpwd` after deployment.** It can change service credentials and make the deployed cloud inconsistent.
4. **Do not restart MariaDB or RabbitMQ with a generic service restart.** Database and broker recovery require cluster-specific evidence.
5. **A restart is a change.** Obtain approval for the exact node and service, check dependencies first, and verify the result afterward.
6. Never paste passwords, tokens, private keys, `clouds.yaml`, or unredacted logs into chat or Git.

For this deployment, the verified OpenStack client cloud is `kolla-admin`, using the user-readable configuration at `$HOME/.config/openstack/clouds.yaml`. The examples set this explicitly so they do not depend on inherited shell state. This lab also has verified passwordless Ansible become; the examples therefore omit `-K`. If become fails on another host, rerun that specific command with `-K` in a visible interactive terminal.

## 0. Start on the management VM

Run this in a new R2D2 terminal:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
export OS_CLOUD=kolla-admin
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"

command -v kolla-ansible
command -v ansible-playbook
hostname
```

**Pass when:** the prompt is on R2D2, both executables resolve inside `$KOLLA_VENV/bin/`, and the inventory path is the current fresh inventory.

**Stop if:** the virtual environment, inventory or target host is wrong.

## 1. Before a planned shutdown

A whole-node shutdown is a cloud outage. Complete these checks before requesting or scheduling poweroff.

### 1A — Record workloads and storage

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
export OS_CLOUD=kolla-admin
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"

openstack server list --all-projects -c ID -c Name -c Status
openstack volume list --all-projects -c ID -c Name -c Status 2>/dev/null || true
```

Record the workloads that must be restored. Shut down guest operating systems through the normal guest method before the host shutdown. A VM in `SHUTOFF` does not by itself prove that guest filesystems or attached storage were cleanly quiesced.

**Stop if:** the API is unavailable, a workload is still active, a volume operation is in progress, or backup and rollback coverage is unknown.

### 1B — Verify console access and exact shutdown targets

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" 'control:network:compute:storage' --list-hosts
```

The output must contain only the physical OpenStack nodes intended for shutdown. The management VM must not appear in the target list.

**Stop if:** an unexpected host appears, an intended node is missing, or console/out-of-band access is unavailable.

### 1C — Run a baseline health check

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
export OS_CLOUD=kolla-admin
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"

openstack token issue -f value -c expires
openstack compute service list
openstack network agent list
ansible -i "$KOLLA_INVENTORY" control -m ansible.builtin.ping -e '{"ansible_become": false}'
```

Record the expected service and agent rows before shutdown. This is a baseline, not proof that every guest, storage backend or cluster is healthy.

## 2. Planned shutdown

There is no verified dependency-ordered whole-cloud shutdown in this guide. The following is a **higher-risk operating-system shutdown fallback**. Use it only after Section 1 passes and after approval for the outage.

Preview the targets again immediately before the command:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" 'control:network:compute:storage' --list-hosts
```

Then schedule a five-minute shutdown on the reviewed physical nodes:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" 'control:network:compute:storage' \
  -b -m ansible.builtin.command \
  -a '/sbin/shutdown -h +5'
```

`CHANGED` means that shutdown was scheduled; it does not mean the host is already powered off. Verify the physical state through the console or out-of-band system. Do not remove power while a host is still shutting down.

If the target review or shutdown result is wrong, try to cancel the scheduled shutdown on the same reviewed targets:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" 'control:network:compute:storage' \
  -b -m ansible.builtin.command \
  -a '/sbin/shutdown -c'
```

Cancellation cannot reach a host that is already stopped or disconnected. Treat uncertain results as a stop condition.

## 3. After powering on the nodes

Power on the physical nodes through their console or approved out-of-band system. Leave R2D2 running. Do not start recovery immediately.

### 3A — Wait for SSH and basic host readiness

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" control -m ansible.builtin.ping \
  -e '{"ansible_become": false}'

ansible -i "$KOLLA_INVENTORY" control -b \
  -m ansible.builtin.command -a 'systemctl is-active docker'

ansible -i "$KOLLA_INVENTORY" control -m ansible.builtin.command \
  -a 'ip -brief address show mgmt0'
```

**Pass when:** all three nodes respond, Docker is active, and each node has its expected management address.

**Stop if:** any node is unreachable, Docker is inactive, or the management interface/address is wrong.

### 3B — Inspect containers without restarting anything

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" control -b \
  -m ansible.builtin.command -a 'docker ps --all --no-trunc'
```

Allow normal startup time. Containers in `starting` state may become healthy. Do not interpret a single early status as a database failure.

### 3C — Run the Kolla readiness gate

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
export ANSIBLE_BECOME_ASK_PASS=false

kolla-ansible prechecks \
  -i "$KOLLA_INVENTORY" \
  --use-test-images
```

A passing precheck is a readiness result. It is not proof that the API, database cluster, broker cluster, workloads or guest connectivity are healthy. This is a full Kolla precheck and may take several minutes on this lab; wait for its final `PLAY RECAP` instead of treating an active terminal as success.

### 3D — Verify API and service heartbeats

Use the actual cloud name from the protected local `clouds.yaml`; do not print credentials or tokens.

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
export OS_CLOUD=kolla-admin
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"

openstack token issue -f value -c expires
openstack compute service list
openstack network agent list
```

**Pass when:** Keystone authentication succeeds, expected compute services are `up` and `enabled`, and expected network agents report alive. Continue to the database and broker checks below.

**If this check fails:** record the failure and continue to Sections 4 and 5 to check Galera and RabbitMQ. An unavailable API can be a consequence of an unhealthy dependency; do not repeat this API check instead of diagnosing the dependency. Do not count intentionally disabled services as required.

## 4. MariaDB/Galera decision after reboot

A MariaDB port timeout or HTTP `502` is only a symptom. Check every configured MariaDB member before selecting recovery.

### 4A — Check containers and port state

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible-inventory -i "$KOLLA_INVENTORY" --graph mariadb
ansible -i "$KOLLA_INVENTORY" mariadb -b \
  -m ansible.builtin.command -a 'docker ps --all --filter name=mariadb'
ansible -i "$KOLLA_INVENTORY" mariadb \
  -m ansible.builtin.shell -a "ss -ltn | grep ':3306 ' || true" \
  -e '{"ansible_become": false}'
```

If containers are still starting, wait and repeat the checks. Do not run recovery just because the first ten-second liveness check failed.

### 4B — Check Galera state on every member

Only run this read-only check after the MariaDB container exists:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" mariadb -b \
  -m ansible.builtin.command \
  -a "docker exec mariadb mariadb --defaults-file=/etc/mysql/healthcheck.cnf --batch --skip-column-names -e \"SHOW GLOBAL STATUS WHERE Variable_name IN ('wsrep_cluster_status','wsrep_cluster_size','wsrep_local_state_comment','wsrep_ready','wsrep_connected','wsrep_cluster_state_uuid','wsrep_incoming_addresses')\""
```

For a single-shard three-node cluster, the expected state is generally:

- `wsrep_cluster_status`: `Primary`
- `wsrep_local_state_comment`: `Synced`
- `wsrep_ready`: `ON`
- `wsrep_connected`: `ON`
- Same non-empty cluster UUID on every member
- Cluster size equal to the configured MariaDB member count

The samples are sequential, not atomic. If a member is missing, unreachable, inconsistent, or the result is unclear, stop.

### 4C — Decide whether recovery is required

| Observed state | Action |
|---|---|
| All members are `Primary`, `Synced`, ready and connected | Do not run recovery. Continue with API, RabbitMQ and workload checks. |
| Containers are starting but ports and health become ready | Wait and recheck. Do not recover. |
| A Primary component may still exist, or member state is uncertain | Do not recover. Investigate every member first. |
| All configured members are reachable, no Primary survives, and the complete outage is confirmed | Review backups, impact and rollback; obtain approval before recovery. |
| A member is missing, logs are stale, shards disagree, or data state is unclear | Stop. Recovery is not justified by the available evidence. |

### 4D — Coordinated MariaDB recovery

This is a change, not a health check. Run only after confirming a complete MariaDB cluster outage, reviewing backups and obtaining approval for this exact action:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
export ANSIBLE_BECOME_ASK_PASS=false
export ANSIBLE_DISPLAY_SKIPPED_HOSTS=false

kolla-ansible mariadb-recovery \
  -b \
  -i "$KOLLA_INVENTORY" \
  --configdir "$KOLLA_CONFIG_PATH" \
  --passwords "$KOLLA_CONFIG_PATH/passwords.yml"
```

The current lab uses passwordless sudo, so this command does not prompt. If become fails, stop; Kolla-Ansible does not accept Ansible's `-K` option. When supported by the installed release, use `ANSIBLE_BECOME_ASK_PASS=true` with a visible terminal prompt instead of the `false` setting above. Never put the password in the guide or chat. Stop on any recovery failure. Do not repeatedly run recovery and do not manually edit `grastate.dat`, delete database volumes, select a bootstrap node, or restart all MariaDB containers independently.

After a successful recovery recap, repeat Section 4B, then check RabbitMQ in Section 5 and the APIs and service heartbeats in Section 5B. Recovery is not complete until the required dependencies and APIs are healthy.

## 5. RabbitMQ check after reboot

A listening broker port does not prove cluster membership or absence of alarms.

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" rabbitmq -b \
  -m ansible.builtin.command \
  -a 'docker exec rabbitmq rabbitmq-diagnostics -q cluster_status'
```

Compare the running members with the `rabbitmq` inventory group. Stop if members are missing, alarms are present, or a network partition is reported. Do not use a generic RabbitMQ restart as a substitute for cluster diagnosis.

### 5A — Verify the RabbitMQ application, not only its container

A running Docker container or active systemd unit does not prove that the RabbitMQ application is running inside the container. Check every RabbitMQ member:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

ansible -i "$KOLLA_INVENTORY" rabbitmq -b \
  -m ansible.builtin.command \
  -a 'docker exec rabbitmq rabbitmq-diagnostics -q check_running'
```

**Pass when:** every expected member reports that RabbitMQ is fully booted and running.

**Stop if:** a member reports that the `rabbit` application is not running. First distinguish a stable container with a stopped application from a container that is repeatedly restarting. On R2D2, sample the state and systemd restart counter twice, 90 seconds apart:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_INVENTORY="$HOME/openstack-fresh/kolla/multinode"
source "$KOLLA_VENV/bin/activate"

ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'docker ps -a --filter name=^/rabbitmq$'
ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'systemctl show kolla-rabbitmq-container.service -p ActiveState -p SubState -p NRestarts'
sleep 90
ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'docker ps -a --filter name=^/rabbitmq$'
ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'systemctl show kolla-rabbitmq-container.service -p ActiveState -p SubState -p NRestarts'
```

An increasing `NRestarts` indicates a restart loop. A counter that does not increase in 90 seconds does not prove long-term stability; compare it over the actual observed restart interval. If the container is cycling, inspect the affected node's RabbitMQ application log and systemd events; do **not** repeat `start_app`, reset the node, or delete its volume. Logs can contain secrets; do not publish them. If the log shows a quorum-queue WAL/segment-writer timeout, the cause and safe remedy must be established before changing broker data.

Only if the container is stable and the Rabbit application is stopped, review `cluster_status`, alarms, partitions and the impact first. With approval for the exact affected node, the targeted action may be:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_INVENTORY="$HOME/openstack-fresh/kolla/multinode"
source "$KOLLA_VENV/bin/activate"

ansible -i "$KOLLA_INVENTORY" node2 -b \
  -m ansible.builtin.command \
  -a 'docker exec rabbitmq rabbitmqctl start_app'
```

Replace `node2` only with the verified affected inventory alias. A successful command is not recovery: recheck container stability, `check_running`, and `cluster_status` on all members. If the container exits or the application stops again, stop and investigate; do not repeat the command.

### 5B — Recheck Placement and dependent API health

Run this after Galera and RabbitMQ are healthy. It detects services that started while a dependency was unavailable and remained unhealthy after the dependency recovered.

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"

ansible -i "$KOLLA_INVENTORY" control -b \
  -m ansible.builtin.command \
  -a 'docker ps --all --filter name=placement_api --no-trunc'

for endpoint in \
  http://192.168.0.150:8780/ \
  http://192.168.0.200:8780/ \
  http://192.168.0.201:8780/ \
  http://192.168.0.202:8780/; do
  curl --silent --show-error --output /dev/null \
    --write-out "$endpoint %{http_code}\n" --max-time 10 "$endpoint"
done

export OS_CLOUD=kolla-admin
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"
openstack compute service list
openstack network agent list
```

**Pass when:** every Placement backend and the VIP return HTTP 200, the Placement containers report healthy, all expected Nova services are `up` and `enabled`, and all expected Neutron agents report alive.

**Stop if:** any Placement backend returns 500/503, a Placement container is unhealthy, or a Nova/Neutron heartbeat is down. If Galera and RabbitMQ already pass, restart only the affected Placement API node(s) one at a time using Section 6, then repeat this check. Do not restart all OpenStack services as a group.

## 6. Restart one individual service

Use this procedure only when a specific service is affected and its dependencies and remaining replicas have been reviewed.

### 6A — Identify the exact service and node

From the management VM, check the API heartbeat first:

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
source "$KOLLA_VENV/bin/activate"
export OS_CLOUD=kolla-admin
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"

openstack compute service list
openstack network agent list
```

Then set the reviewed inventory alias and Kolla systemd unit. Do not type `NODE_ALIAS` literally:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

NODE_ALIAS=node1
SERVICE_UNIT=kolla-neutron_openvswitch_agent-container.service

ansible -i "$KOLLA_INVENTORY" "$NODE_ALIAS" -b \
  -m ansible.builtin.command \
  -a "systemctl show $SERVICE_UNIT -p LoadState -p ActiveState -p SubState"
```

Confirm that the unit exists and that the selected container is present before requesting a restart.

Common service-to-unit examples:

| Service | Kolla systemd unit |
|---|---|
| Nova compute | `kolla-nova_compute-container.service` |
| Nova API | `kolla-nova_api-container.service` |
| Nova conductor | `kolla-nova_conductor-container.service` |
| Placement API | `kolla-placement_api-container.service` |
| Neutron Open vSwitch agent | `kolla-neutron_openvswitch_agent-container.service` |
| Neutron L3 agent | `kolla-neutron_l3_agent-container.service` |
| Neutron DHCP agent | `kolla-neutron_dhcp_agent-container.service` |
| Neutron metadata agent | `kolla-neutron_metadata_agent-container.service` |
| Horizon | `kolla-horizon-container.service` |
| HAProxy | `kolla-haproxy-container.service` |

Unit names can differ when services are not enabled or the installed Kolla release changes. Verify the exact unit on the target node.

### 6B — Restart only the approved service

A restart interrupts the selected service and can start a stopped unit. Run only after approval for the exact `NODE_ALIAS` and `SERVICE_UNIT`:

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

NODE_ALIAS=node1
SERVICE_UNIT=kolla-neutron_openvswitch_agent-container.service

ansible -i "$KOLLA_INVENTORY" "$NODE_ALIAS" -b \
  -m ansible.builtin.command \
  -a "systemctl restart $SERVICE_UNIT"
```

Do **not** use this generic procedure for MariaDB or RabbitMQ. Use Sections 4 and 5 instead.

### 6C — Verify the restart

```bash
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"

NODE_ALIAS=node1
SERVICE_UNIT=kolla-neutron_openvswitch_agent-container.service

ansible -i "$KOLLA_INVENTORY" "$NODE_ALIAS" -b \
  -m ansible.builtin.command \
  -a "systemctl show $SERVICE_UNIT -p LoadState -p ActiveState -p SubState"
```

Then repeat the relevant API heartbeat check and inspect the selected container. A successful `systemctl restart` only proves that systemd accepted the request; it does not prove application health.

## 7. Recovery completion checklist

Recovery is complete only when all applicable checks pass:

- [ ] All intended nodes respond to Ansible.
- [ ] `mgmt0` has the expected address on every node.
- [ ] Docker is active on every node.
- [ ] Expected Kolla containers are running and healthy.
- [ ] `kolla-ansible prechecks` passes.
- [ ] Keystone authentication succeeds.
- [ ] MariaDB members are `Primary`, `Synced`, ready and connected.
- [ ] RabbitMQ members match inventory with no alarms or partitions, and the RabbitMQ application is running on every member.
- [ ] Placement VIP and every expected Placement backend return HTTP 200.
- [ ] Compute services report `up` and `enabled`.
- [ ] Network agents report alive and enabled.
- [ ] Horizon/API VIP responds if Horizon is enabled.
- [ ] Previously recorded workloads are restored only after the control plane and dependencies are healthy.

A passing container check, API check or systemd check alone is not end-to-end cloud health.
