Introducing disk health checks for Elasticsearch (ES) in your on-premises ELK Stack at OVHcloud’s Poland data center is a critical enhancement to ensure reliability, especially given your setup’s scale (~10TB logs/day, 3 master nodes, 5 data nodes, 2 coordinating nodes). I’ll integrate disk health monitoring into the existing workflow, set up alerts for proactive intervention, and detail the process to replace a failing disk on a data node while replicating data ASAP to minimize downtime and data loss. This will fit seamlessly into your hybrid environment (VMware, bare metal, EKS) with Fluent Bit, Kafka, Logstash, Kibana, and ILM (90-day hot-warm-cold-delete), leveraging tools like Prometheus, Grafana, Terraform, Ansible, and Vault. I’ll maintain the granular, real-time narrative style you’ve appreciated, ensuring you can explain this confidently in an interview.

---

### Context Recap
Your Elasticsearch cluster runs on 10 physical servers in Poland:
- **3 Master Nodes**: Coordinate cluster state (`es-master01-03`, 500GB SSD).
- **5 Data Nodes**: Store logs:
  - 2 Hot (`es-data01-02`, 4TB NVMe, fresh logs).
  - 3 Warm (`es-data03-05`, 8TB HDD, older logs).
- **2 Coordinating Nodes**: Route queries (`es-coord01-02`, 500GB SSD).
Logs from CRM, ERP, EKS (payment, order processing), Nginx, and MySQL are processed at ~100k events/sec, indexed into ~100GB/day indices, managed by ILM. Disk health is critical, as data nodes store primary and replica shards, and a disk failure risks data loss or downtime.

---

### Step 1: Introducing Disk Health Checks for Elasticsearch

#### Why Monitor Disk Health?
Elasticsearch is disk-intensive—hot nodes handle high IOPS for indexing, warm nodes store large volumes. A failing disk (e.g., SMART errors, high latency, or corruption) can:
- Cause shard unavailability, turning the cluster `yellow` (no replicas) or `red` (no primaries).
- Slow indexing, spiking Logstash backlogs.
- Risk data loss if replicas aren’t healthy.

We’ll monitor SMART metrics, disk I/O, and ES-specific disk usage to catch issues early.

#### Implementation
We’ll use **Prometheus** and **node_exporter** for disk health, **Elasticsearch metrics** for usage, and **Grafana** for visualization/alerting, all running on-prem in Poland.

1. **Node Exporter for SMART and I/O**:
   - Deploy `node_exporter` on each data node (`es-data01-05`) to scrape disk metrics.
   - Ansible playbook:
     ```yaml
     - name: Install node_exporter
       hosts: es_data_nodes
       tasks:
         - name: Install node_exporter
           apt:
             name: prometheus-node-exporter
             state: present
         - name: Configure node_exporter
           copy:
             content: |
               --collector.diskstats
               --collector.smart
             dest: /etc/systemd/system/node-exporter.service.d/override.conf
         - name: Restart node_exporter
           systemd:
             name: prometheus-node-exporter
             state: restarted
     ```
   - Metrics:
     - `node_smartmon_health`: SMART status (0 = OK, 1 = failing).
     - `node_disk_io_time_seconds_total`: Time spent on I/O (high = slow disk).
     - `node_filesystem_avail_bytes`: Free disk space.

2. **Elasticsearch Metrics**:
   - Use `elasticsearch-exporter` (already in use) for ES-specific disk metrics.
   - Metrics:
     - `elasticsearch_filesystem_data_available_bytes`: Free space per node.
     - `elasticsearch_filesystem_data_used_percent`: Usage percentage.
     - `elasticsearch_indices_store_size_bytes`: Index size per node.

3. **Prometheus Config**:
   - Scrape `node_exporter` (port 9100) and `elasticsearch-exporter` (port 9114).
   - Config (`/etc/prometheus/prometheus.yml`):
     ```yaml
     scrape_configs:
       - job_name: 'es_data_nodes'
         static_configs:
           - targets:
               - 'es-data01.pl.ovh.local:9100'
               - 'es-data02.pl.ovh.local:9100'
               - 'es-data03.pl.ovh.local:9100'
               - 'es-data04.pl.ovh.local:9100'
               - 'es-data05.pl.ovh.local:9100'
         metrics_path: /metrics
       - job_name: 'es_metrics'
         static_configs:
           - targets:
               - 'es-data01.pl.ovh.local:9114'
               - 'es-data02.pl.ovh.local:9114'
               - 'es-data03.pl.ovh.local:9114'
               - 'es-data04.pl.ovh.local:9114'
               - 'es-data05.pl.ovh.local:9114'
         metrics_path: /metrics
     ```

4. **Grafana Dashboard**:
   - Create a “ES Disk Health” dashboard:
     - **Panel 1**: SMART Status (`node_smartmon_health{device=~”nvme0n1|sda”}`).
     - **Panel 2**: Disk I/O Time (`rate(node_disk_io_time_seconds_total[5m])`).
     - **Panel 3**: ES Disk Usage (`elasticsearch_filesystem_data_used_percent`).
     - **Panel 4**: Free Space (`node_filesystem_avail_bytes`).
   - Query Example:
     ```promql
     100 - (node_filesystem_avail_bytes{instance=~”es-data.*”,mountpoint=”/data”} / node_filesystem_size_bytes{instance=~”es-data.*”,mountpoint=”/data”} * 100)
     ```

5. **Alerts**:
   - Define in Prometheus (`/etc/prometheus/alerts.yml`):
     ```yaml
     groups:
     - name: es_disk_health
       rules:
       - alert: ESDiskSMARTFailure
         expr: node_smartmon_health{instance=~”es-data.*”} > 0
         for: 5m
         labels:
           severity: critical
         annotations:
           summary: “Disk SMART failure on {{ $labels.instance }}”
           description: “Disk {{ $labels.device }} is reporting failure. Replace ASAP.”
       - alert: ESDiskHighIO
         expr: rate(node_disk_io_time_seconds_total{instance=~”es-data.*”}[5m]) > 0.5
         for: 10m
         labels:
           severity: warning
         annotations:
           summary: “High disk I/O on {{ $labels.instance }}”
           description: “Disk {{ $labels.device }} is slow, impacting performance.”
       - alert: ESDiskLowSpace
         expr: elasticsearch_filesystem_data_available_bytes{instance=~”es-data.*”} < 500e9
         for: 5m
         labels:
           severity: critical
         annotations:
           summary: “Low disk space on {{ $labels.instance }}”
           description: “Less than 500GB free. Risk of shard allocation failure.”
     ```
   - Route to PagerDuty and Slack:
     ```yaml
     receivers:
       - name: 'pagerduty'
         pagerduty_configs:
           - service_key: "${PAGERDUTY_KEY}"
       - name: 'slack'
         slack_configs:
           - api_url: "${SLACK_WEBHOOK}"
             channel: '#monitoring'
     ```

**Granular Flow**:
- `node_exporter` on `es-data01` detects SMART error on `/dev/nvme0n1` (`node_smartmon_health=1`).
- Prometheus scrapes every 15s, evaluates `ESDiskSMARTFailure` rule.
- After 5m, alerts PagerDuty (“Disk failure on es-data01”) and Slack.
- Grafana plots `node_smartmon_health` spike, `elasticsearch_filesystem_data_used_percent` at 80%.

**Why This?**:
- SMART catches physical failures (wear, bad sectors).
- I/O metrics flag performance issues (latency > 500ms).
- ES metrics prevent allocation failures (`disk.watermark.high: 90%`).

---

### Step 2: Replacing a Failing Disk and Replicating Data

#### Scenario
Let’s say `es-data01` (hot node, 4TB NVMe) triggers `ESDiskSMARTFailure` at 10:00 UTC, April 11, 2025. It hosts primary shards for `crm.app-2025.04.11`, `nginx.access-2025.04.11`, and replicas for `eks.payment-2025.04.11`. We need to replace the disk and replicate data ASAP to avoid downtime or data loss.

#### Process
Replacing a disk on an on-prem physical server involves hardware swaps, OS reconfiguration, and Elasticsearch shard reallocation, all while maintaining cluster health.

1. **Preparation**:
   - **Confirm Failure**: SSH to `es-data01` (`ssh es-data01.pl.ovh.local`):
     ```bash
     smartctl -a /dev/nvme0n1
     ```
     - Output: `Critical Warning: 0x02 (Temperature), Reallocated_Sector_Ct: 100`.
   - **Check Cluster**: Query health:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X GET "https://es-master01.pl.ovh.local:9200/_cluster/health?pretty"
     ```
     - Status: `green` (replicas on `es-data02-05` are intact).
   - **Exclude Node**: Prevent new shard allocations to `es-data01`:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X PUT "https://es-master01.pl.ovh.local:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
     {
       "transient": {
         "cluster.routing.allocation.exclude._host": "es-data01.pl.ovh.local"
       }
     }'
     ```
   - **Verify Shards**: List shards:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X GET "https://es-master01.pl.ovh.local:9200/_cat/shards?v"
     ```
     - Primaries on `es-data01` (e.g., `crm.app-2025.04.11 shard 1`) have replicas elsewhere.

2. **Evacuate Data**:
   - **Relocate Shards**: Manually move primaries to other hot nodes:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X POST "https://es-master01.pl.ovh.local:9200/_cluster/reroute" -H 'Content-Type: application/json' -d'
     {
       "commands": [
         {
           "move": {
             "index": "crm.app-2025.04.11",
             "shard": 1,
             "from_node": "es-data01",
             "to_node": "es-data02"
           }
         },
         {
           "move": {
             "index": "nginx.access-2025.04.11",
             "shard": 3,
             "from_node": "es-data01",
             "to_node": "es-data02"
           }
         }
       ]
     }'
     ```
   - **Monitor Relocation**:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X GET "https://es-master01.pl.ovh.local:9200/_cat/recovery?v"
     ```
     - Shows ~100GB moving to `es-data02`, ETA 10 minutes at ~200MB/s.
   - **Prometheus Metric**: `elasticsearch_cluster_shard_active` drops on `es-data01`.
   - **Impact**: Indexing continues (Logstash targets `es-data02`), Kibana queries hit replicas.

3. **Shut Down Node**:
   - **Stop Elasticsearch**: Ensure no active shards:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X GET "https://es-data01.pl.ovh.local:9200/_cat/shards?v"
     ```
     - Empty for `es-data01`.
   - Stop service:
     ```bash
     systemctl stop elasticsearch
     ```
   - **Power Down Server**:
     - Coordinate with OVHcloud data center team in Warsaw.
     - Use iLO/IPMI: `ipmitool -H es-data01-mgmt.pl.ovh.local -U admin -P ${IPMI_PASS} power off`.

4. **Replace Disk**:
   - **Physical Swap**:
     - Data center tech removes failing NVMe (`/dev/nvme0n1`).
     - Installs new 4TB NVMe (same model, e.g., Samsung PM1735).
     - Time: ~15 minutes (Warsaw staff on-site).
   - **Verify Hardware**:
     - Power on server: `ipmitool power on`.
     - Check BIOS: Confirm new disk detected.
     - Boot Ubuntu 20.04, verify:
       ```bash
       lsblk
       # Output: nvme0n1 4T
       smartctl -a /dev/nvme0n1
       # Output: Overall-health: PASSED
       ```

5. **Reconfigure Disk**:
   - **Format and Mount**:
     - Partition: `fdisk /dev/nvme0n1`, create GPT, single partition (`nvme0n1p1`).
     - Filesystem: `mkfs.ext4 /dev/nvme0n1p1`.
     - Mount: Update `/etc/fstab`:
       ```fstab
       /dev/nvme0n1p1 /data ext4 defaults 0 2
       ```
     - Mount: `mount /data`.
   - **Restore ES Path**:
     - Recreate directory: `mkdir /data/elasticsearch`.
     - Set permissions: `chown elasticsearch:elasticsearch /data/elasticsearch`.
   - **Ansible Automation**:
     ```yaml
     - name: Configure new disk on es-data01
       hosts: es-data01.pl.ovh.local
       tasks:
         - name: Partition disk
           command: echo -e "g\nn\n1\n\n\nw" | fdisk /dev/nvme0n1
         - name: Format disk
           filesystem:
             fstype: ext4
             dev: /dev/nvme0n1p1
         - name: Mount disk
           mount:
             path: /data
             src: /dev/nvme0n1p1
             fstype: ext4
             state: mounted
         - name: Create ES directory
           file:
             path: /data/elasticsearch
             state: directory
             owner: elasticsearch
             group: elasticsearch
             mode: '0755'
     ```

6. **Rejoin Cluster**:
   - **Start Elasticsearch**:
     - Update config (`/etc/elasticsearch/elasticsearch.yml`):
       ```yaml
       path.data: /data/elasticsearch
       path.logs: /var/log/elasticsearch
       ```
     - Start service: `systemctl start elasticsearch`.
   - **Verify Join**:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X GET "https://es-master01.pl.ovh.local:9200/_cat/nodes?v"
     ```
     - Shows `es-data01` with `roles: data_hot`.
   - **Remove Exclusion**:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X PUT "https://es-master01.pl.ovh.local:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
     {
       "transient": {
         "cluster.routing.allocation.exclude._host": null
       }
     }'
     ```

7. **Replicate Data**:
   - **Shard Allocation**:
     - Masters automatically assign replica shards to `es-data01` (ILM policies prioritize hot nodes).
     - Example: `crm.app-2025.04.11 shard 1` replica moves from `es-data03` to `es-data01`.
   - **Force Allocation** (if slow):
     ```bash
     curl -u elastic:${ES_PASSWORD} -X POST "https://es-master01.pl.ovh.local:9200/_cluster/reroute" -H 'Content-Type: application/json' -d'
     {
       "commands": [
         {
           "allocate_replica": {
             "index": "crm.app-2025.04.11",
             "shard": 1,
             "node": "es-data01"
           }
         }
       ]
     }'
     ```
   - **Monitor**:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X GET "https://es-master01.pl.ovh.local:9200/_cat/recovery?v"
     ```
     - Shows ~100GB replicating at ~300MB/s, done in ~6 minutes.
   - **Prometheus**: `elasticsearch_cluster_shard_active` rises on `es-data01`.

8. **Verify Cluster**:
   - Check health:
     ```bash
     curl -u elastic:${ES_PASSWORD} -X GET "https://es-master01.pl.ovh.local:9200/_cluster/health?pretty"
     ```
     - Status: `green`, `active_shards: 1000`.
   - Query Kibana: `crm.app-* level:ERROR` returns recent logs.
   - Grafana: `elasticsearch_filesystem_data_used_percent` on `es-data01` climbs normally.

**Timeline**:
- 10:00 UTC: Alert triggers.
- 10:05: Confirm failure, exclude node.
- 10:15: Relocate shards.
- 10:25: Shut down node.
- 10:40: Replace disk.
- 10:50: Reconfigure disk.
- 11:00: Rejoin cluster, replicate data.
- 11:10: Cluster `green`. Total: ~70 minutes.

**Granular Flow**:
- **Alert**: Prometheus detects `node_smartmon_health=1` on `es-data01`, notifies PagerDuty.
- **Evacuation**: Shards move to `es-data02`, Logstash redirects indexing.
- **Swap**: Tech replaces NVMe, Ansible formats `/data`.
- **Replication**: `es-data01` rejoins, masters assign replicas, data copies at ~300MB/s.
- **Validation**: Kibana queries hit `es-data01` shards, Grafana shows healthy IOPS.

**Why This?**:
- **Exclusion**: Prevents new writes to failing disk, keeps cluster `green`.
- **Relocation**: Moves primaries proactively, replicas ensure no data loss.
- **Ansible**: Automates disk setup, minimizes errors.
- **Fast Replication**: NVMe’s ~3GB/s bandwidth speeds recovery.

---

### Integration with ELK Workflow
- **Fluent Bit**: Unaffected, ships to Kafka.
- **Kafka**: Buffers logs during `es-data01` downtime, Logstash consumes post-recovery.
- **Logstash**: Targets `es-data02-05` while `es-data01` is down, no backlog due to Kafka.
- **Elasticsearch**: Replicas on warm nodes (`es-data03-05`) serve reads, ILM continues (e.g., `crm.app-2025.04.04` moves to warm).
- **Kibana**: Queries hit `es-coord01-02`, users see no disruption.

**Monitoring**:
- Prometheus: `elasticsearch_cluster_shard_unassigned` spikes during relocation, drops post-recovery.
- Grafana: Plots `node_disk_io_time_seconds_total` to confirm new disk’s performance.
- Alert Cleared: `ESDiskSMARTFailure` resolves when `node_smartmon_health=0`.

**ILM Impact**:
- Hot node (`es-data01`) outage doesn’t affect warm/cold phases.
- New indices (e.g., `eks.payment-2025.04.11`) start on `es-data02`, shift to `es-data01` post-recovery.

---

### Operational Considerations
- **Spares**: Keep 2 NVMe drives in Warsaw for rapid swaps.
- **Testing**: Post-replacement, run `fio` to benchmark disk:
  ```bash
  fio --name=write_test --filename=/data/test --size=10G --rw=write --bs=4M
  ```
- **Documentation**: Update runbook in Confluence (`ELK_Disk_Replacement.md`).
- **Training**: Drill team quarterly on disk swaps via OVHcloud’s lab.

**Failure Modes**:
- **Slow Replication**: Increase `indices.recovery.max_bytes_per_sec: 500mb`.
- **Node Won’t Join**: Check `elasticsearch.yml`, verify certs (`/etc/elasticsearch/certs`).
- **Data Loss**: Restore from snapshot if replicas fail:
  ```bash
  curl -u elastic:${ES_PASSWORD} -X POST "https://es-master01.pl.ovh.local:9200/_snapshot/elk_backup/daily_2025-04-10/_restore" -H 'Content-Type: application/json' -d'
  {
    "indices": "crm.app-2025.04.11"
  }'
  ```

**Optimizations**:
- Pre-allocate shards to `es-data02` during high load:
  ```bash
  curl -u elastic:${ES_PASSWORD} -X PUT "https://es-master01.pl.ovh.local:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
  {
    "transient": {
      "cluster.routing.allocation.include._host": "es-data02.pl.ovh.local"
    }
  }'
  ```
- Use `async` recovery for large shards:
  ```bash
  curl -u elastic:${ES_PASSWORD} -X POST "https://es-master01.pl.ovh.local:9200/_cluster/reroute?async=true"
  ```

---

### Interview-Ready Explanation
“Our ELK Stack in OVHcloud’s Poland data center processes 10TB of logs daily. To ensure disk health on our 5 data nodes, we monitor SMART status, I/O latency, and free space with Prometheus’s `node_exporter` and `elasticsearch-exporter`. Alerts trigger via PagerDuty for SMART failures (`node_smartmon_health > 0`) or low space (<500GB), visualized in Grafana. If a disk fails—like an NVMe on `es-data01`—we exclude the node, relocate shards to `es-data02` using `_cluster/reroute`, and power down via IPMI. A tech swaps the disk in ~15 minutes, Ansible formats it (`ext4`, mounts `/data`), and the node rejoins the cluster. Masters assign replicas, replicating ~100GB in ~6 minutes at 300MB/s. The cluster stays `green` with replicas on warm nodes, Logstash redirects via Kafka, and Kibana remains responsive. This keeps our 90-day ILM policy and 100+ users unaffected.”

---

This integrates disk health checks and replacement into your ELK workflow with granular steps. If you want more—like specific SMART metrics, deeper recovery tweaks, or a mock incident—let me know, and I’ll dive in! How’s this for your interview prep? Ready to rock it?
