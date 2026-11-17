# Shutdown, Reboot and Recovery

**Purpose:** shut the three-node Kolla-Ansible lab down cleanly, bring it back, and fix only what is actually broken.

This is the only recovery guide for this lab. The older `openstack-from-zero` guide describes a different deployment — do not run its scripts here.

## Lab facts

| Item | Value |
|---|---|
| Management VM | `r2d2` |
| Kolla venv | `$HOME/.venvs/kolla-fresh` |
| Kolla config | `$HOME/openstack-fresh/kolla` |
| Inventory | `$HOME/openstack-fresh/kolla/multinode` |
| API VIP | `192.168.0.150` |
| Nodes | `node1` (.200), `node2` (.201), `node3` (.202) |
| OpenStack cloud | `kolla-admin` |

### Service ports (memorise these)

| Service | Port | Healthy |
|---|---|---|
| Keystone | 5000 | `300` |
| Glance | 9292 | `300` |
| Nova API | 8774 | `200` |
| Neutron | 9696 | `200` |
| **Placement** | **8780** | `200` |
| Horizon | 80 | `302` |

> **Placement is 8780, not 8778.** Probing 8778 always returns `000` — that is not a failure.

## TL;DR — is the lab healthy?

```bash
for p in 5000 9292 8774 9696 8780 80; do printf "%s: " $p; curl -s -o /dev/null -m6 -w '%{http_code}\n' http://192.168.0.150:$p/; done
```

All `200`/`300`/`302` → healthy. Anything else → work down the check ladder in Part 2, fix the first failure, run this again.

## Symptom → action

| Symptom | Action |
|---|---|
| A node does not ping | Console access. Host is down. |
| VIP on zero nodes, or on two | Keepalived/HAProxy problem — Part 2, Layer 2. |
| API returns `000` or `503` | Service is not listening. Check the container and its dependencies. |
| API returns `500` but container says `healthy` | **Stale worker.** Restart that one service. Part 3. |
| Placement 500, Nova 503, Neutron 503 | Typical after a reboot where the DB came up late. Part 3. |
| `systemctl` says `active`, API still 500 | Same stale-worker case. `systemctl` is not proof. |
| RabbitMQ container cycling | Stop and read the log. Do not loop `start_app`. Part 3C. |
| Cinder check "fails" | Cinder is not deployed here. Expected. |

## Safety rules

1. **Health checks first.** A failed port check is a symptom, not proof recovery is needed.
2. **A reboot does not require recovery.** Give the hosts time to settle, then verify.
3. **Never run `kolla-genpwd` after deployment.** It changes credentials.
4. **Never restart MariaDB, RabbitMQ, ProxySQL, HAProxy or Keepalived generically.** They need cluster-specific evidence.
5. **A restart is a change.** Approve the exact node and service, check dependencies, verify after.
6. **Never paste passwords, tokens, keys, `clouds.yaml` or unredacted logs into chat or Git.**

The golden rule: **if a service started before the database was writable, restart it — it will never fix itself.**

## Session setup (run once per terminal)

```bash
export KOLLA_VENV="$HOME/.venvs/kolla-fresh"
export KOLLA_CONFIG_PATH="$HOME/openstack-fresh/kolla"
export KOLLA_INVENTORY="$KOLLA_CONFIG_PATH/multinode"
source "$KOLLA_VENV/bin/activate"
export OS_CLOUD=kolla-admin
export OS_CLIENT_CONFIG_FILE="$HOME/.config/openstack/clouds.yaml"
```

Every command below assumes this block has run. Passwordless sudo is configured, so no `-K` is needed. If become fails, stop.

---

# Part 1 — Planned shutdown

A whole-lab shutdown is an outage. Complete all three steps first.

## 1.1 Record workloads

```bash
openstack server list --all-projects -c ID -c Name -c Status
openstack volume list --all-projects -c ID -c Name -c Status 2>/dev/null || true
```

Shut guest OSes down properly first. `SHUTOFF` alone does not prove filesystems were quiesced.

**Stop if:** the API is down, a volume operation is in flight, or backup coverage is unknown.

## 1.2 Confirm the shutdown targets

```bash
ansible -i "$KOLLA_INVENTORY" 'control:network:compute:storage' --list-hosts
```

The list must contain only the three OpenStack nodes. **`r2d2` must not appear.**

**Stop if:** an unexpected host appears, a node is missing, or console access is unavailable.

## 1.3 Baseline health check

```bash
openstack token issue -f value -c expires
openstack compute service list
openstack network agent list
ansible -i "$KOLLA_INVENTORY" control -m ansible.builtin.ping -e '{"ansible_become": false}'
```

Record the expected rows. This is a baseline, not proof that every guest and storage backend is healthy.

## 1.4 Schedule the shutdown

Re-run the target list from 1.2 immediately before this command. Then:

```bash
ansible -i "$KOLLA_INVENTORY" 'control:network:compute:storage' \
  -b -m ansible.builtin.command -a '/sbin/shutdown -h +5'
```

`CHANGED` means the shutdown was *scheduled*, not that the host is off. Confirm the physical state on the console. Never cut power while a host is shutting down.

To cancel on the same reviewed targets:

```bash
ansible -i "$KOLLA_INVENTORY" 'control:network:compute:storage' \
  -b -m ansible.builtin.command -a '/sbin/shutdown -c'
```

Cancellation cannot reach a host that is already down. Treat an uncertain result as a stop condition.

---

# Part 2 — Power-on and the check ladder

Power the nodes on via console. Leave `r2d2` running. **Do not start repairing immediately.**

Work top to bottom. **Stop at the first red layer.** Never fix a lower layer while an upper one is broken.

## Layer 1 — Hosts reachable

```bash
ansible -i "$KOLLA_INVENTORY" control -m ansible.builtin.ping -e '{"ansible_become": false}'
ansible -i "$KOLLA_INVENTORY" control -b -m ansible.builtin.command -a 'systemctl is-active docker'
ansible -i "$KOLLA_INVENTORY" control -m ansible.builtin.command -a 'ip -brief address show mgmt0'
```

**Pass:** 3/3 respond, Docker active, `mgmt0` addresses correct.
**Fail:** a node is unreachable or Docker is inactive → console access.

## Layer 2 — VIP owned by exactly one node

```bash
for h in 200 201 202; do printf "node: "; ssh joe@192.168.0.$h 'ip -4 -o addr show | grep -c 192.168.0.150'; done
```

**Pass:** exactly one `1`.
**Fail:** zero everywhere → VIP lost. Two → split brain. Do not restart HAProxy/Keepalived until you know which.

## Layer 3 — Containers (inspect only, restart nothing)

```bash
ansible -i "$KOLLA_INVENTORY" control -b -m ansible.builtin.command -a 'docker ps --all --no-trunc'
```

Allow startup time. `starting` is normal early on. Ignore containers with no healthcheck by design: `keepalived`, `fluentd`, `cron`, `kolla_toolbox`, `neutron_metadata_agent`, dhcp agents.

**The real signal is the restart count:**

```bash
ansible -i "$KOLLA_INVENTORY" control -b -m ansible.builtin.command \
  -a 'docker inspect --format "{{.Name}} {{.RestartCount}}" $(docker ps -aq)'
```

A climbing count = crash loop. A `healthy` label means nothing on its own.

## Layer 4 — Database (Galera) — check before anything else

```bash
ansible -i "$KOLLA_INVENTORY" mariadb -b -m ansible.builtin.command \
  -a 'docker ps --all --filter name=mariadb'
```

Then read the cluster state on every member:

```bash
ansible -i "$KOLLA_INVENTORY" mariadb -b -m ansible.builtin.command \
  -a "docker exec mariadb mariadb --defaults-file=/etc/mysql/healthcheck.cnf --batch --skip-column-names -e \"SHOW GLOBAL STATUS WHERE Variable_name IN ('wsrep_cluster_status','wsrep_cluster_size','wsrep_local_state_comment','wsrep_ready','wsrep_connected','wsrep_cluster_state_uuid','wsrep_incoming_addresses')\""
```

**Pass:** all members `Primary`, `Synced`, `ready=ON`, `connected=ON`, same non-empty UUID, size = 3.

Then confirm it is actually **writable** — this is the check that matters:

```bash
for h in 200 201 202; do ssh joe@192.168.0.$h 'sudo docker exec placement_api timeout 4 bash -c "cat </dev/null >/dev/tcp/192.168.0.150/3306" && echo "DB OK" || echo "DB FAIL"'; done
```

### Recovery decision

| Observed | Action |
|---|---|
| All `Primary`/`Synced`/ready/connected | **Do not recover.** Go to Layer 5. |
| Containers starting, ports becoming ready | Wait and recheck. Do not recover. |
| A Primary may still exist, or state is unclear | Do not recover. Investigate every member. |
| All members reachable, no Primary, outage confirmed | Backups + approval first, then `kolla-ansible mariadb-recovery`. |
| A member missing, stale logs, shards disagree | **Stop.** Evidence does not justify recovery. |

Only after a confirmed complete outage, reviewed backups and explicit approval:

```bash
kolla-ansible mariadb-recovery -b -i "$KOLLA_INVENTORY" \
  --configdir "$KOLLA_CONFIG_PATH" --passwords "$KOLLA_CONFIG_PATH/passwords.yml"
```

Never edit `grastate.dat`, delete volumes, hand-pick a bootstrap node, or restart all MariaDB containers independently. Stop on any failure and do not re-run recovery.

## Layer 5 — RabbitMQ (2/3 quorum is enough to run)

```bash
ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'docker exec rabbitmq rabbitmq-diagnostics -q cluster_status'
```

**Pass:** members match inventory, no alarms, no partitions. A port being open proves nothing.

Then check the **application**, not the container:

```bash
ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'docker exec rabbitmq rabbitmq-diagnostics -q check_running'
```

**Pass:** every member reports "fully booted and running".
**Fail:** a member reports the app is not running → distinguish *stopped* from *cycling*:

```bash
ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'docker ps -a --filter name=^/rabbitmq$'
ansible -i "$KOLLA_INVENTORY" rabbitmq -b -m ansible.builtin.command \
  -a 'systemctl show kolla-rabbitmq-container.service -p ActiveState -p SubState -p NRestarts'
sleep 90
# repeat both commands and compare NRestarts
```

- **NRestarts rising** → crash loop. Read the log. Do **not** loop `start_app`, reset the node or delete the volume.
- **Stable container, app stopped** → review `cluster_status`, alarms and impact first. With approval for that exact node only:

```bash
ansible -i "$KOLLA_INVENTORY" node2 -b -m ansible.builtin.command \
  -a 'docker exec rabbitmq rabbitmqctl start_app'
```

Replace `node2` only with the verified alias. Then recheck stability, `check_running` and `cluster_status`. If it stops again, stop and investigate — do not repeat.

**Known limitation:** a quorum-queue WAL/segment-writer timeout has no verified one-command repair here. Preserve the broker data; never delete the volume or reset the cluster as a shortcut. With 2/3 members running and no alarms, the cloud works — leave the stuck member alone until you have evidence.

## Layer 6 — APIs and service heartbeats

```bash
for p in 5000 9292 8774 9696 8780 80; do printf "%s: " $p; curl -s -o /dev/null -m6 -w '%{http_code}\n' http://192.168.0.150:$p/; done
for ip in 200 201 202; do printf "node %s 8780: " $ip; curl -s -o /dev/null -m6 -w '%{http_code}\n' http://192.168.0.$ip:8780/; done
openstack compute service list
openstack network agent list
```

**Pass:** all endpoints `200`/`300`/`302`, every Placement backend `200`, all Nova services `up`+`enabled`, all network agents alive.

**Fail:** a `500` from a container that reports `healthy` means **stale workers** — restart that service (Part 3). Do not re-run the API check instead of diagnosing the dependency.

---

# Part 3 — Repair

## 3A — Restart one service

Verify the unit exists first:

```bash
NODE_ALIAS=node1
SERVICE_UNIT=kolla-placement_api-container.service
ansible -i "$KOLLA_INVENTORY" "$NODE_ALIAS" -b -m ansible.builtin.command \
  -a "systemctl show $SERVICE_UNIT -p LoadState -p ActiveState -p SubState"
```

Then, with approval for that exact node and unit:

```bash
ansible -i "$KOLLA_INVENTORY" "$NODE_ALIAS" -b -m ansible.builtin.command \
  -a "systemctl restart $SERVICE_UNIT"
```

If the unit hit its start limit: `systemctl reset-failed $SERVICE_UNIT` first.

Then verify — a successful restart only proves systemd accepted it:

```bash
ansible -i "$KOLLA_INVENTORY" "$NODE_ALIAS" -b -m ansible.builtin.command \
  -a "systemctl show $SERVICE_UNIT -p ActiveState -p SubState -p NRestarts"
```

Follow with the matching endpoint check from Layer 6.

### Unit names

| Service | Unit |
|---|---|
| Nova compute | `kolla-nova_compute-container.service` |
| Nova API | `kolla-nova_api-container.service` |
| Nova conductor | `kolla-nova_conductor-container.service` |
| Nova scheduler | `kolla-nova_scheduler-container.service` |
| Placement API | `kolla-placement_api-container.service` |
| Neutron server | `kolla-neutron_server-container.service` |
| Neutron OVS agent | `kolla-neutron_openvswitch_agent-container.service` |
| Neutron L3 agent | `kolla-neutron_l3_agent-container.service` |
| Neutron DHCP agent | `kolla-neutron_dhcp_agent-container.service` |
| Neutron metadata agent | `kolla-neutron_metadata_agent-container.service` |
| Horizon | `kolla-horizon-container.service` |
| HAProxy | `kolla-haproxy-container.service` |

Unit names change between Kolla releases. Always verify on the target node.

## 3B — The stale-worker fix (most common post-reboot fault)

If the hosts booted before MariaDB was writable, these containers keep serving `500` while `systemctl` says `active` and Docker says `healthy`. Restart them, one node at a time:

```bash
for u in placement_api nova_api nova_conductor nova_scheduler neutron_server; do
  for h in 200 201 202; do
    ssh joe@192.168.0.$h "sudo systemctl reset-failed kolla-${u}-container.service 2>/dev/null; sudo systemctl restart kolla-${u}-container.service"
  done
done
```

Then re-run Layer 6. Conductor and scheduler may be in a restart loop (`NRestarts` in the hundreds) — `reset-failed` clears the limit so they can start.

## 3C — Run the Kolla readiness gate

```bash
kolla-ansible prechecks -i "$KOLLA_INVENTORY" --use-test-images
```

Wait for the final `PLAY RECAP`. A passing precheck is a readiness result, not proof of cloud health. It takes several minutes on this lab.

---

# Part 4 — Completion checklist

Recovery is complete only when all of these pass:

- [ ] 3/3 nodes respond to Ansible
- [ ] `mgmt0` has the expected address on every node
- [ ] Docker is active on every node
- [ ] VIP is on exactly one node
- [ ] MariaDB 3/3 `Primary`, `Synced`, ready, connected
- [ ] DB is writable via VIP:3306
- [ ] RabbitMQ members match inventory, no alarms, no partitions, app running
- [ ] Placement VIP and all backends return `200`
- [ ] Nova API `200`, all Nova services `up` + `enabled`
- [ ] Neutron `200`, all network agents alive
- [ ] Horizon responds if enabled
- [ ] `kolla-ansible prechecks` passes
- [ ] Workloads restored only after the control plane is healthy
- [ ] Guest boot, storage and network tested separately — untested means untested

A passing container, API or systemd check alone is not end-to-end cloud health.
