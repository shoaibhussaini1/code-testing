# Reverse KT: OpenShift Cluster Services (Kafka, Elastic, MongoDB, Monitoring, Logging)

As of 2026-11-18 · Author: Shoaib

## 1. Purpose and how to use this guide

This guide explains how the services on the OpenShift 4.22 cluster are deployed, run, and alerted on, so the team can operate them without the current owner. Read sections 2 and 3 first, then the section for each tool. Section 13 lists commands that confirm what is actually running, and section 14 is the inventory to fill in during the session.

Where a detail depends on your cluster's exact versions or custom resources, it is marked **verify**. Those items are the questions to close during the KT.

## 2. Cluster snapshot

The cluster runs OpenShift **4.22.20**. The `oc get clusterversion` output on the as-of date shows AVAILABLE True and PROGRESSING False, so no upgrade is running.

- SINCE 20d: the cluster has reported this state for 20 days.
- STATUS: "Cluster version is 4.22.20".
- Run `oc get co` next. Every ClusterOperator should show AVAILABLE True, PROGRESSING False, DEGRADED False.

## 3. How services are deployed on this cluster

Most services run as an **Operator** that watches custom resources (CRs). You change the CR, and the Operator changes the pods, services, and config. Few things are deployed as bare YAML. Helm and GitOps are the other two routes. Each row below is the likely route. Confirm it with the command in the last column.

| Service | Usual route on OpenShift | Objects to look at | Confirm with |
| --- | --- | --- | --- |
| Kafka | Strimzi or AMQ Streams operator, installed by OLM | Kafka, KafkaTopic, KafkaUser, KafkaConnect | `oc get kafka -A` |
| Elasticsearch and Kibana | Elastic Cloud on Kubernetes (ECK) operator | Elasticsearch, Kibana, Beat | `oc get elasticsearch -A` |
| MongoDB | MongoDB Community or Enterprise Kubernetes Operator | MongoDBCommunity or MongoDB | `oc get mongodbcommunity -A` |
| Mongosync | Deployment or Job running the mongosync binary | Deployment, ConfigMap, Secret | `oc get deploy -A \| findstr mongosync` |
| Prometheus and Alertmanager | Built-in cluster monitoring, plus user workload monitoring | Prometheus, Alertmanager, ServiceMonitor, PrometheusRule | `oc -n openshift-monitoring get pods` |
| Grafana | Grafana operator or a Deployment; dashboards as CRs or ConfigMaps | Grafana, GrafanaDashboard, GrafanaDatasource | `oc get grafana -A` |
| Dynatrace | Dynatrace Operator with a DynaKube CR | DynaKube, OneAgent DaemonSet, ActiveGate | `oc -n dynatrace get dynakube` |
| Splunk | Splunk OpenTelemetry Collector or Splunk Connect, through Helm | DaemonSet, Deployment, Helm release | `helm list -A` |
| Fluent Bit | Helm chart or DaemonSet owned by the team | DaemonSet, ConfigMap | `oc get ds -A \| findstr fluent` |

Namespace names are not in this table on purpose. Take them from the discovery output in section 13.

**verify:** whether any of these are managed by Argo CD from Git. If so, change the Git repo, not the cluster, or the change will be reverted.

## 4. Kafka

Kafka is most likely run by the Strimzi or AMQ Streams operator. The operator turns a `Kafka` CR into brokers, services, and config. Topics and users are their own CRs, so they can live in Git.

**What to check**

- `oc get kafka -A`, then `oc describe kafka <name> -n <ns>`. The Ready condition must be True.
- `oc get kafkatopic -n <ns>` for topic partitions and replication factor.
- `oc get kafkauser -n <ns>` for users and ACLs. Never edit secrets by hand.
- Broker pods: `oc get pods -n <ns> -l strimzi.io/cluster=<name>`. All should be Running and Ready.
- Broker disk: check the PVC usage in Grafana before it fills.

**Maintenance**

- Operator upgrades arrive through OLM. Check whether the Subscription is set to manual or automatic approval.
- Kafka version upgrades are a `spec.kafka.version` change. Check the operator's support matrix first.
- Rolling restarts are handled by the operator. Do not delete broker pods to restart them.

**Typical alerts** (names vary; check the PrometheusRule)

- Under-replicated partitions above 0: a broker is slow, down, or lost its disk.
- Offline partitions above 0: a partition has no leader. Treat as urgent.
- Active controller count not equal to 1: a controller election problem.
- Consumer lag growing: a consumer is slow or stuck. Check the consumer app first.
- Disk or PVC above 80 percent: add storage or reduce retention.

**Found in the cluster:** Kafka runs in ZooKeeper mode, not KRaft. The `wallcare-cluster` resource is in `dev-app-sup-services` and `qa-app-sup-services`. Each has 5 Kafka replicas and 5 ZooKeeper replicas, and both report READY True.

**Open:** WARNINGS is True in both namespaces. Run `oc describe kafka wallcare-cluster -n dev-app-sup-services` and read the conditions. Kafka 4.0 no longer supports ZooKeeper, so plan any move to KRaft as a separate project. **verify** the Strimzi version and which Kafka Connect or MirrorMaker clusters exist.

## 5. Elasticsearch

Elasticsearch is most likely run by the Elastic Cloud on Kubernetes (ECK) operator. An `Elasticsearch` CR defines the version, node sets, and storage. A `Kibana` CR, if present, sits beside it. The operator performs rolling upgrades and restarts.

**What to check**

- `oc get elasticsearch -A` shows HEALTH (green, yellow, red) and PHASE. Expect green and Ready.
- `oc get pods -n <ns> -l elasticsearch.k8s.elastic.co/cluster-name=<name>` for node pods.
- Passwords live in the secret `<name>-es-elastic-user`. Read them when needed. Do not paste them into chats or docs.
- Cluster health from a workstation: `oc port-forward svc/<name>-es-http 9200 -n <ns>`, then query `_cluster/health`.

**Health meaning**

- Green: all primary and replica shards are assigned.
- Yellow: primaries are fine, but some replicas are not assigned. Often a node is missing or disk is short.
- Red: at least one primary shard is unavailable. Treat as an incident. Search and ingest may fail.

**Maintenance**

- Version upgrades are a `spec.version` change. Check the ECK and Elasticsearch compatibility table first.
- Disk watermarks: at the flood stage, indices become read-only. Add storage before it reaches that point.
- Snapshots: confirm the snapshot repository and its schedule. **verify** which team owns it.

**Typical alerts** (names vary; check the PrometheusRule or the Splunk and Elastic alerts)

- Cluster status red or yellow for more than 15 minutes.
- Node down, or fewer data nodes than expected.
- JVM heap above 85 percent for a sustained period.
- Disk above the high watermark.
- Ingest or search rejections rising.

**verify:** which indices are business-critical, and who consumes them.

## 6. MongoDB and Mongosync

**MongoDB** is most likely run by a MongoDB Kubernetes Operator. The Community operator uses a `MongoDBCommunity` CR. The Enterprise operator uses a `MongoDB` CR and connects to Ops Manager or Cloud Manager. Each replica set is a set of pods, named `<name>-0`, `<name>-1`, and so on.

- `oc get mongodbcommunity -A` (or `oc get mongodb -A` for Enterprise). PHASE should be Running.
- Pod roles: `oc get pods -n <ns>` shows the pods. Inside a pod, `rs.status()` in mongosh shows which member is PRIMARY.
- Users and passwords come from secrets or a MongoDBUser CR. Do not edit them by hand.

**Mongosync** moves or syncs data between two clusters. It runs as a process with an HTTP API, usually on port 27182 inside its pod. Its states include IDLE, RUNNING, PAUSED, COMMITTED, and FAILED. **verify** the exact states for your Mongosync version.

- Check progress through the API endpoint for progress, or the pod logs.
- Source and destination credentials are in a Secret or ConfigMap. Confirm which one.
- **Committing a Mongosync sync cuts over to the destination. Do not run commit without the owner's sign-off.** Confirm the cutover plan first.

**Typical alerts** (names vary)

- Replica set has no PRIMARY for more than 1 minute.
- Replication lag above the agreed threshold.
- Mongosync pod restarting, or state FAILED.
- Mongosync lag not shrinking across several checks.
- Disk or connection count near limits.

**Maintenance**

- Operator upgrades follow OLM. MongoDB version upgrades are a `spec.version` change, applied one member at a time by the operator.
- Mongosync must be a version the source and destination both support. Check the compatibility page before upgrading.

**verify:** backup method and owner, and whether the Mongosync job is still active or left over from a past migration.

## 7. Prometheus, Alertmanager, and the alert path

Metrics flow from the app to Prometheus. Alerts flow from Prometheus to people. Each step is a separate object you can inspect:

1. A `ServiceMonitor` or `PodMonitor` tells Prometheus which endpoints to scrape.
2. Prometheus scrapes those endpoints on a fixed interval and stores the samples.
3. A `PrometheusRule` holds the alert expressions. An alert turns Pending, then Firing, once its `for:` duration has passed.
4. Firing alerts go to Alertmanager, which groups, deduplicates, silences, and routes them.
5. Alertmanager sends each alert to a receiver such as email, Slack, PagerDuty, or a ticket queue. **verify** which receivers are configured.

**Where to look**

- Console: Observe, then Alerting. It shows firing and pending alerts with the rule behind each one.
- `oc get prometheusrule -A` lists the rule objects. `oc get servicemonitor -A` lists the scrape targets.
- Platform monitoring runs in `openshift-monitoring`. User workload monitoring runs in `openshift-user-workload-monitoring`. **verify** which one owns each rule.
- Silences live in Alertmanager. A quiet alert is not proof of health if someone silenced it.

**How to triage an alert**

- Read the alert's summary and runbook annotations first.
- Open the expression in Prometheus and see whether the condition is still true.
- Check the labels to find the namespace, pod, or cluster that is affected.

## 8. Grafana

Grafana reads data from data sources and shows it on dashboards. It does not store metrics itself. If a dashboard is empty, check the data source before the dashboard.

- Instance: `oc get grafana -A` if the Grafana operator is used, or `oc get deploy -A | findstr grafana` for a plain Deployment.
- Access: `oc get route -A | findstr grafana` gives the URL.
- Data sources: `oc get grafanadatasource -A`, or the Data sources page in the UI. Use the Test button on each one.
- Dashboards: stored as `GrafanaDashboard` CRs or as ConfigMaps with a label. Keep them in Git if the team does so.

**Alerting in Grafana is a separate system.** Grafana-managed alerts are evaluated by Grafana, not by Prometheus. When an alert fires, find out which system owns it before you start triage.

**Maintenance**

- Data source credentials are in secrets. Rotate them with the owning team, not by editing the dashboard.
- Plugin and Grafana version upgrades follow the operator or the Deployment image. Test on a dashboard before upgrading.

**verify:** which data sources are in use (likely Prometheus, Elasticsearch, Dynatrace), and who owns the Grafana admin role.

## 9. Dynatrace

Dynatrace is installed by its Operator, which reads a `DynaKube` custom resource. The DynaKube says which Dynatrace tenant to report to, which monitoring mode to use, and where the ActiveGate runs. The operator is usually in a namespace named `dynatrace`. **verify**.

**What to check**

- `oc -n dynatrace get dynakube`. The phase should be Running.
- `oc -n dynatrace get pods` for the operator, ActiveGate, and any webhook pods.
- `oc -n dynatrace get ds` for OneAgent. It runs one pod per node, and every node should be covered.
- The API token is in a secret. Do not paste it anywhere.

**How it works**

- OneAgent collects host and process data. In cloud-native mode, an injection webhook and CSI driver add the agent to application pods.
- If pods fail to start with injection errors after a Dynatrace change, check the webhook and the CSI driver first.

**Alerts**

- Dynatrace raises problems inside the Dynatrace tenant. Those problems reach people through alerting profiles or integrations, such as Slack or ServiceNow. **verify** which integrations are set up.
- Some alerts exist only in Dynatrace, not in Prometheus. Look in both places before assuming an alert is missing.

**Maintenance**

- Operator upgrades follow OLM or Helm. Check the DynaKube for auto-update settings.
- Rotate the API tokens before they expire, and update the secret. Do not recreate the DynaKube just to rotate.

**verify:** which monitoring mode is in use, and the Dynatrace owner.

## 10. Splunk

Splunk receives data from the cluster through an HTTP Event Collector (HEC) endpoint. A collector in the cluster sends logs to that endpoint. The collector is either the Splunk OpenTelemetry Collector or Splunk Connect for Kubernetes, usually installed with Helm. **verify** which one.

**Data path:** container logs, then the collector (often fed by Fluent Bit), then the Splunk HEC endpoint, then an index.

**What to check in the cluster**

- `helm list -A | findstr splunk` shows the release and its chart version.
- `oc get ds,deploy -A | findstr splunk` shows the collector pods.
- Collector logs: `oc logs ds/<collector> -n <ns>`. Look for HEC errors.

**Common HEC responses**

- 401: the HEC token is invalid or was rotated. Update the secret the collector reads.
- 503: Splunk is busy or the queue is full. Check Splunk indexing health with the Splunk admin.
- Data missing but pods healthy: check the index and sourcetype names with the Splunk admin.

**What to check in Splunk**

- Confirm the expected indexes and that events are arriving.
- A saved search or alert for missing data is the best early warning. Ask whether one exists.

**Maintenance**

- Rotate the HEC token with the Splunk admin, then update the secret in the cluster.
- Chart upgrades use Helm. Read the release notes for changed value names.

**verify:** index names, the daily ingest limit, and who owns index retention.

## 11. Fluent Bit and the logging pipeline

Fluent Bit reads container logs on every node and forwards them to the log backends. Each node runs one Fluent Bit pod, so a node without one sends no logs.

[Diagram: App pods (stdout and stderr) → Fluent Bit (one pod per node) → Splunk (HEC endpoint and index) and Elasticsearch (indexes, viewed in Kibana). verify which streams go to which backend, and by which tags.]

Read the drawing left to right. Logs leave the pods, pass through Fluent Bit, and land in one or both backends.

**What to check**

- `oc get ds -A | findstr fluent` shows the DaemonSet. DESIRED should equal READY.
- `oc get cm -A | findstr fluent` shows the config. Read the output section first, since it names the backend.
- `oc logs ds/<name> -n <ns>` shows errors. Look for retry or connection-refused lines.
- Check the backend's health before blaming Fluent Bit.

**Maintenance**

- Change the ConfigMap or the Helm values, not the running pods. Restart the DaemonSet after a config change if the chart does not do it.
- Read the release notes before a version upgrade. Plugin and parser names change between versions.

**Typical alerts:** output errors rising, DaemonSet READY below DESIRED, or no logs from a namespace.

**verify:** whether this cluster also runs the OpenShift logging operator. Recent OpenShift logging versions use Vector, not Fluent Bit. Two collectors on one node would ship every log twice.

## 12. Day-2 maintenance

Most of the routine work is watching alerts and renewing credentials. Upgrades come last, and they are planned one service at a time.

**Daily, on-call**

- [ ] Review firing alerts in the OpenShift console, Dynatrace, and Splunk.
- [ ] Run `oc get co` and confirm every ClusterOperator is Available and not Degraded.
- [ ] Check the health of Kafka, Elasticsearch, and MongoDB, and the Fluent Bit and Dynatrace DaemonSets.

**Weekly**

- [ ] Check PVC usage for Kafka, Elasticsearch, MongoDB, and Prometheus.
- [ ] Run `oc get installplan -A` and review any operator upgrade waiting for approval.
- [ ] Review silences in Alertmanager. Expire the ones with no owner.

**Monthly, or before a change**

- [ ] Check expiry dates for the Splunk HEC token, the Dynatrace API token, and TLS certificates.
- [ ] Confirm the last backups and Elasticsearch snapshots completed. **verify** the backup owner.
- [ ] Check each operator's compatibility with the target OpenShift version before any cluster upgrade.

**Upgrade order.** Upgrade the cluster only after each operator confirms support for the new version. Change one stateful service at a time, and keep a rollback plan for each.

## 13. Discovery commands

Run these before the session. They fill in section 14 without anyone having to remember the names. Commands work the same in PowerShell, except the filter step, shown in the second block.

```bash
# Platform health and versions
oc get clusterversion
oc get co

# Operators installed through OLM, and pending upgrades
oc get csv -A
oc get subscriptions -A
oc get installplan -A

# Custom resources for each service
oc get crd | grep -Ei 'kafka|elastic|mongo|prometheus|alertmanager|grafana|dynakube|fluent|vector|clusterlogforwarder'

# Namespaces and pods for each service
oc get ns | grep -Ei 'kafka|elastic|mongo|monitor|grafana|dynatrace|splunk|logging|fluent'
oc get pods -A -o wide | grep -Ei 'kafka|elastic|mongo|grafana|dynatrace|splunk|fluent|mongosync'

# DaemonSets (OneAgent, Fluent Bit, Splunk collector) and Helm releases
oc get ds -A
helm list -A

# Alerting and scrape config
oc get prometheusrule -A
oc get servicemonitor -A
oc get alertmanagerconfig -A
```

```powershell
# PowerShell: the filter step uses Select-String instead of grep
oc get pods -A -o wide | Select-String -Pattern 'kafka|elastic|mongo|grafana|dynatrace|splunk|fluent'

# Alertmanager routing config (contains receiver URLs; do not paste it into chats)
$v = oc -n openshift-monitoring get secret alertmanager-main -o jsonpath='{.data.alertmanager\.yaml}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($v))
```

If a command returns nothing, the service is probably in a different namespace or not installed. Record that in section 14 as a finding, not a gap.

## 14. Inventory to fill in

Fill one row per service during the session. Leave a cell blank if nobody knows yet, and add it to the open questions in section 15.

| Service | Namespace | Deployed by | Version | Owner | Alerts go to |
| --- | --- | --- | --- | --- | --- |
| Kafka | dev-app-sup-services, qa-app-sup-services (cluster `wallcare-cluster`) | Strimzi operator, ZooKeeper mode | | | |
| Elasticsearch | | | | | |
| Kibana | | | | | |
| MongoDB | | | | | |
| Mongosync | | | | | |
| Prometheus | openshift-monitoring | | | | |
| Alertmanager | openshift-monitoring | | | | |
| Grafana | | | | | |
| Dynatrace | | | | | |
| Splunk collector | | | | | |
| Fluent Bit | | | | | |

## 15. KT session plan and open questions

A 90-minute session covers the platform first, then the services, then the alerts. Run the discovery commands before the session starts.

1. Cluster snapshot and platform health, about 10 minutes (sections 2 and 13).
2. How services are deployed, and the inventory, about 15 minutes (sections 3 and 14).
3. Service walkthroughs for Kafka, Elasticsearch, MongoDB and Mongosync, Dynatrace, and Splunk, about 30 minutes.
4. The alert path. Open three firing alerts together and triage each one, about 15 minutes.
5. The logging path through Fluent Bit, about 10 minutes.
6. On-call routine, upgrades, and open questions, about 10 minutes.

**Open questions to close during the session**

- [ ] Which services are managed by Helm or Argo CD, and where is the Git repo for each?
- [ ] Is Fluent Bit deployed directly, and is the OpenShift logging operator also running?
- [ ] Is any Mongosync job still active, and who approves a cutover?
- [ ] Which alerts are silenced, and who created each silence?
- [ ] Where are the alert receivers defined, and who gets paged for each one?
- [ ] Which Splunk indexes are in use, and when does the HEC token expire?
- [ ] Who owns the MongoDB backups and the Elasticsearch snapshots?
- [ ] Which Grafana dashboards are business-critical?
- [ ] Who holds cluster-admin and namespace access for each service?
