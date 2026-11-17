# RabbitMQ queue cleanup

**Purpose:** inspect RabbitMQ queues; a safe, tested quorum-queue deletion procedure is not yet available in this guide. Run these commands on a verified node hosting the `rabbitmq` Docker container, not the management VM.

**STOP: do not use the earlier deletion command.** This lab's quorum queues reject `--if-unused`; simply removing safeguards can lose messages. No quorum-queue deletion command in this guide has been tested or approved.

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

## 3. Stop before deletion

The previous command failed for this lab's quorum queues and has been removed. For the large number of Heat queues, first establish their purpose, message and consumer state, recovery coverage, and the reason they accumulated; then review and approve a quorum-compatible, rate-limited cleanup plan. If a previous delete failed or timed out, inspect the exact queue before retrying because a timeout alone does not prove whether deletion occurred.
