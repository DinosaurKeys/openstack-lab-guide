# RabbitMQ queue cleanup

**Purpose:** inspect RabbitMQ queues and delete one specifically reviewed queue. Run these commands on a verified node hosting the `rabbitmq` Docker container, not the management VM. This procedure is not a bulk purge or a RabbitMQ recovery procedure.

**Quorum-queue warning:** the lab's RabbitMQ rejects `--if-unused` for quorum queues, and conditional deletion may also reject `--if-empty`; do not use the deletion block below for quorum queues or assume those flags protect them. Stop and review a separate quorum-queue deletion plan: an unconditional delete has no atomic empty/unused safeguard and can lose messages.

**Stop before deleting** if the queue's owner or virtual host is unclear, the broker is unhealthy or partitioned, messages or consumers are present, or a service may need the queue. Zero messages and consumers do not alone prove a queue is obsolete. Deleting a queue cannot be undone merely by recreating its name: bindings, policies and messages may differ or be lost. Establish suitable recoverable backup coverage and an approved rollback before a destructive cleanup; do not remove broker volumes or reset the cluster.

## 1. Check RabbitMQ

```bash
sudo docker exec rabbitmq rabbitmq-diagnostics -q check_running
sudo docker exec rabbitmq rabbitmqctl cluster_status
sudo docker exec rabbitmq rabbitmqctl list_vhosts
```

**Pass when:** the broker is fully booted, all expected nodes are running, and there are no alarms or network partitions. Identify the actual virtual host from the last command; `/` is common but must be confirmed.

## 2. Inspect queues

The following uses virtual host `/`. If your confirmed vhost differs, stop and replace `/` in **each** command with the exact vhost. Listing a large number of queues can take time and put load on the broker; do not repeat a stalled listing blindly.

```bash
sudo docker exec rabbitmq rabbitmqctl list_queues -p / \
  name type consumers messages messages_ready messages_unacknowledged
```

Find the exact queue name, type, owner and whether its service is still in use. Heat queue prefixes can help identify candidates, but prefixes are not proof that a queue is obsolete. Investigate `messages`, `messages_ready`, `messages_unacknowledged`, and `consumers`; stop if any are nonzero or unavailable. Confirm the service's workload, queue purpose, bindings and recovery plan before proceeding. Disabling Heat for a fresh deployment is covered in [Deploy OpenStack](02-deploy-openstack.md#optional-disable-heat-before-a-fresh-deployment).

## 3. Delete one approved queue

**Classic queues only:** this permanently deletes the queue. Stop if the queue is quorum or its type is unknown. Select **one exact name** from the inspected output; do not use wildcards or `--force`. Enter it at the prompt on the node; the value is kept only in this shell session.

```bash
read -r -p 'Exact approved queue name: ' QUEUE
if [ -z "$QUEUE" ]; then
  printf '%s\n' 'STOP: no queue name supplied.'
else
  printf 'Target: vhost=/ queue=%s\n' "$QUEUE"
  read -r -p 'Type DELETE to confirm: ' CONFIRM
  if [ "$CONFIRM" = DELETE ]; then
    sudo docker exec rabbitmq rabbitmqctl delete_queue -p / "$QUEUE" --if-empty --if-unused
  else
    printf '%s\n' 'Cancelled; nothing deleted.'
  fi
fi
```

The broker checks for an empty queue and no consumers at deletion time; these flags do not establish ownership or future need. If deletion fails or times out, stop and inspect before retrying: the request may have partially succeeded. For tens of thousands of candidates, do not loop this command blindly; create and review a separately approved, rate-limited cleanup plan with monitoring and recovery steps.

## 4. Verify

Rerun the queue listing from Step 2 and confirm the exact queue is absent. Then rerun Step 1 and verify the affected OpenStack service's API and functionality. A successful delete command alone does not establish cluster or workload health. If the queue reappears, investigate its producer instead of repeatedly deleting it.
