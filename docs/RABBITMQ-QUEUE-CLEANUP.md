# RabbitMQ queue cleanup

**Purpose:** inspect and delete one confirmed-empty Heat quorum queue. Run these commands on a verified node hosting the `rabbitmq` Docker container, not the management VM.

**Quorum-queue rule:** this lab rejects `--if-unused` for quorum queues. Do **not** use `--if-unused`, `--if-empty`, or `--force` for the Heat queues below. RabbitMQ supports ordinary queue deletion for quorum queues, but the deletion itself is permanent.

**Stop before deleting** if the queue's owner or virtual host is unclear, the broker is unhealthy or partitioned, messages or consumers are present, or a service may need the queue. Zero messages and consumers do not alone prove a queue is obsolete. Deleting a queue cannot be undone merely by recreating its name: bindings, policies and messages may differ or be lost. Establish suitable recoverable backup coverage and an approved rollback before a destructive cleanup; do not remove broker volumes or reset the cluster.

## 1. Check RabbitMQ

```bash
sudo docker exec rabbitmq rabbitmq-diagnostics -q check_running
sudo docker exec rabbitmq rabbitmqctl cluster_status
sudo docker exec rabbitmq rabbitmqctl list_vhosts
```

**Pass when:** the broker is fully booted, all expected nodes are running, and there are no alarms or network partitions. Identify the actual virtual host from the last command; `/` is common but must be confirmed.

## 2. Show five Heat queue candidates

The following uses virtual host `/`. If your confirmed vhost differs, stop and replace `/` in **each** command with the exact vhost. This command stops after five matching queues; do not run an unrestricted `list_queues` query against this 44,000-queue broker.

```bash
sudo timeout 30 docker exec rabbitmq rabbitmqctl list_queues -p / name type consumers messages -q \
  | grep -m 5 -E '^(heat-engine-listener|engine_worker)\.'
```

Each row is `queue-name<TAB>type<TAB>consumers<TAB>messages`. Use only a row whose last three values are exactly `quorum`, `0`, and `0`. Copy **only the first field**—for example, `heat-engine-listener.f6146e92-3e86-4bc0-8c10-75a1910e430c`—as the queue name. Stop if the type is not `quorum`, or consumers or messages are nonzero. Disabling Heat for a fresh deployment is covered in [Deploy OpenStack](02-deploy-openstack.md#optional-disable-heat-before-a-fresh-deployment).

## 3. Delete exactly one confirmed-empty Heat quorum queue

Paste the complete first field from Step 2 when prompted. This command deliberately omits `--if-unused` because RabbitMQ rejects that conditional operation for quorum queues. A successful `Deleted:` line means the broker accepted deletion of that one queue.

```bash
read -r -p 'Paste one confirmed-empty Heat queue name: ' QUEUE
case "$QUEUE" in
  heat-engine-listener.*|engine_worker.*) ;;
  *) printf '%s\n' 'STOP: not a Heat queue name.'; exit 1 ;;
esac

sudo docker exec rabbitmq rabbitmqctl delete_queue -p / "$QUEUE" --timeout 60 &&
  printf 'Deleted: %s\n' "$QUEUE"
```

Do not convert this into a loop or run it in parallel. Delete one queue, check that RabbitMQ remains healthy with Step 1, then decide whether to continue. A full cleanup of tens of thousands of queues needs a separately reviewed, rate-limited procedure and live monitoring.
