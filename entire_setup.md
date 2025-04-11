Thank you for pushing for an even deeper, granular understanding of the entire ELK Stack workflow. As an ELK administrator at OVHcloud, managing an on-premises setup in Poland (Warsaw) for a hybrid environment (VMware VMs, bare metal, AWS EKS) with 15+ projects, I’ll dive into the minutiae of every component—Fluent Bit, Kafka, Logstash, Elasticsearch, Kibana, and their integrations—while addressing your specified setup: 3 master nodes, 5 data nodes, 2 coordinating nodes, and a hot-warm-cold-delete ILM policy with 90-day retention. I’ll trace the log flow from source to visualization with exhaustive detail, covering configurations, failure modes, optimizations, and operational tasks, ensuring you can articulate this at an expert level in an interview. I’ll also revisit log sources (CRM, ERP, EKS microservices, Nginx, MySQL) and tie in observability tools (Prometheus, Grafana, Jaeger, Vault). Buckle up—this will be comprehensive!

---

### System Context
Our ELK Stack is the observability cornerstone for OVHcloud’s hybrid cloud, processing ~10TB of logs daily from monolithic apps (CRM, ERP), EKS microservices (payment, order processing), and infrastructure (Nginx, MySQL). Hosted on physical servers in Poland for sovereignty, it serves 100+ users with real-time insights. The stack integrates Fluent Bit for collection, Kafka for buffering, Logstash for processing, Elasticsearch for storage, and Kibana for visualization, with ILM enforcing a 90-day hot-warm-cold-delete lifecycle. Prometheus/Grafana monitors health, Jaeger correlates traces, and Vault secures credentials, all managed via Terraform and Ansible.

#### Node Architecture
- **3 Master Nodes**: Cluster coordination, metadata management.
- **5 Data Nodes**: Log storage/indexing (2 hot, 3 warm).
- **2 Coordinating Nodes**: Query routing, performance optimization.

---

### Granular Workflow: End-to-End Log Flow

#### 1. Log Sources and Collection with Fluent Bit
**Role**: Fluent Bit is our ultra-lightweight log forwarder (~450KB memory), deployed on every log source to collect, tag, and ship logs to Kafka. It’s chosen for efficiency, critical for EKS pods (thousands of containers) and resource-constrained VMs.

**Log Sources**:
- **CRM (VMware, Germany)**:
  - Path: `/var/log/crm/app.log`.
  - Format: Text (`ERROR: User login failed - ID: 123`).
  - Volume: ~1GB/day, ~10k events/minute.
- **ERP (Bare Metal, Poland)**:
  - Path: `/opt/erp/logs/erp.log`.
  - Format: JSON (`{"event": "inventory_updated", "sku": "ABC123"}`).
  - Volume: ~500MB/day, ~5k events/minute.
- **Payment Service (EKS, eu-central-1)**:
  - Path: Container `stdout/stderr`.
  - Format: JSON (`{"user_id": "456", "event": "payment_processed"}`).
  - Volume: ~2GB/day, ~20k events/minute.
- **Order Processing (EKS)**:
  - Path: Container `stdout`.
  - Format: JSON (`{"order_id": "789", "status": "shipped"}`).
  - Volume: ~1.5GB/day, ~15k events/minute.
- **Nginx (VMware)**:
  - Path: `/var/log/nginx/access.log`, `/var/log/nginx/error.log`.
  - Format: Apache Combined (`192.168.1.1 - - [10/Apr/2025:10:00:00 +0000] "GET /api" 200`).
  - Volume: ~1GB/day, ~50k events/minute.
- **MySQL (Bare Metal)**:
  - Path: `/var/log/mysql/mysql.log`.
  - Format: Text (`2025-04-10 10:00:00 ERROR: Deadlock detected`).
  - Volume: ~200MB/day, ~2k events/minute.

**Fluent Bit Deployment**:
- **VMware/Bare Metal**: Installed via Ansible (`ansible-playbook -i inventory fluent-bit.yml`).
  - Service: `/etc/systemd/system/fluent-bit.service`.
  - Config: `/fluent-bit/etc/fluent-bit.conf`.
- **EKS**: DaemonSet in `logging` namespace, managed by Argo CD.
  - ConfigMap: `fluent-bit-config`.
  - Image: `fluent/fluent-bit:2.1`.

**Config Details** (VMware):
```ini
[SERVICE]
    Flush         5             # Flush every 5s
    Log_Level     info
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020        # Health check endpoint

[INPUT]
    Name          tail
    Path          /var/log/crm/app.log
    Tag           crm.app
    Read_from_Head True
    Refresh_Interval 10

[INPUT]
    Name          tail
    Path          /var/log/nginx/access.log
    Tag           nginx.access
    Parser        nginx

[FILTER]
    Name          modify
    Match         *
    Add           hostname ${HOSTNAME}
    Add           region poland
    Add           env prod

[FILTER]
    Name          record_modifier
    Match         crm.app
    Record        source vmware-crm

[OUTPUT]
    Name          kafka
    Match         *
    Brokers       kafka.pl.ovh.local:9092
    Topics        raw-logs
    Retry_Limit   3
    rdkafka.queue.buffering.max.messages 10000
    rdkafka.batch.num.messages 1000
    rdkafka.linger.ms 5
```

**EKS Config** (ConfigMap):
```yaml
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name          tail
        Path          /var/log/containers/*.log
        Tag           eks.*
        Parser        docker
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    [FILTER]
        Name          kubernetes
        Match         eks.*
        Merge_Log     On
        K8S-Logging.Parser On
        Labels        Off
        Annotations   Off

    [OUTPUT]
        Name          kafka
        Match         *
        Brokers       kafka.pl.ovh.local:9092
        Topics        raw-logs
        Retry_Limit   3
  parsers.conf: |
    [PARSER]
        Name          docker
        Format        json
        Time_Key      time
        Time_Format   %Y-%m-%dT%H:%M:%S.%L
```

**Parsers** (`parsers.conf`):
```ini
[PARSER]
    Name          nginx
    Format        regex
    Regex         ^(?<clientip>[^ ]+) - - \[(?<timestamp>[^\]]+)\] "(?<request>[^"]+)" (?<status>\d+) (?<size>\d+)
    Time_Key      timestamp
    Time_Format   %d/%b/%Y:%H:%M:%S %z

[PARSER]
    Name          json
    Format        json
    Time_Key      timestamp
    Time_Format   %Y-%m-%dT%H:%M:%S
```

**Granular Flow**:
- **CRM Log**: Fluent Bit tails `/var/log/crm/app.log`, matches `ERROR: User login failed`.
- **Parsing**: Uses `regex` for Nginx, `json` for EKS, adds `hostname`, `env: prod`.
- **Buffering**: Queues up to 10k messages internally, batches 1k to Kafka.
- **Shipping**: Sends to `raw-logs` topic, retries 3 times on failure.
- **EKS Nuance**: Kubernetes filter enriches with pod metadata (`namespace`, `pod_name`).
- **Failure Mode**: If Kafka’s down, Fluent Bit buffers to disk (`/fluent-bit/storage/`), backpressures to avoid loss.

**Why This?**: Fluent Bit’s low CPU (~0.1 core) suits EKS scale, regex/json parsers handle diverse formats, and Kafka output ensures reliability. We monitor via Prometheus (`fluentbit_output_proc_bytes_total`).

---

#### 2. Kafka Buffering
**Role**: Kafka (3 brokers, physical servers in Poland) buffers logs to decouple Fluent Bit from Logstash, handling spikes and ensuring no loss during ELK maintenance.

**Deployment**:
- Servers: `kafka01-03.pl.ovh.local`, 12 CPU, 64GB RAM, 2TB SSD.
- Managed: Ansible (`ansible-playbook kafka.yml`).
- Version: Kafka 3.6.

**Config Details**:
- **Broker** (`server.properties`):
  ```properties
  broker.id=1
  listeners=PLAINTEXT://kafka01.pl.ovh.local:9092
  num.partitions=16
  default.replication.factor=2
  log.retention.hours=24
  min.insync.replicas=2
  message.max.bytes=10485760
  ```
- **Topic**:
  ```bash
  kafka-topics.sh --create --topic raw-logs --partitions 16 --replication-factor 2 --bootstrap-server kafka.pl.ovh.local:9092
  ```

**Monitoring**:
- Prometheus: `kafka_topic_partition_leader` (broker health), `kafka_consumer_lag` (Logstash backlog).
- Grafana: Dashboard for partition skew, message rate.

**Granular Flow**:
- **Input**: Fluent Bit sends a CRM log (`ERROR: User login failed`) to `raw-logs` partition 5.
- **Storage**: Kafka replicates across 2 brokers (e.g., `kafka01`, `kafka02`), commits offset.
- **Retention**: Keeps logs 24 hours, ~500GB total.
- **Output**: Logstash consumes via `group_id: logstash`, reading all 16 partitions.
- **Failure Mode**: If a broker dies, replication ensures availability; Logstash re-reads uncommitted offsets.
- **Optimization**: `linger.ms=5` batches messages, `batch.num.messages=1000` reduces network overhead.

**Why Kafka?**: Buffers EKS spikes (e.g., 100k events/sec during sales), allows Logstash restarts without loss. 16 partitions scale for ~10TB/day, 2 replicas balance reliability vs. storage.

---

#### 3. Logstash Processing
**Role**: Logstash (2 servers, 16 CPU, 64GB RAM) parses, transforms, and enriches logs, routing them to Elasticsearch with structured indices.

**Deployment**:
- Servers: `logstash01-02.pl.ovh.local`.
- Managed: Ansible (`ansible-playbook logstash.yml`).
- Version: Logstash 8.10.

**Config Details**:
```conf
input {
  kafka {
    bootstrap_servers => "kafka.pl.ovh.local:9092, kafka02.pl.ovh.local:9092"
    topics => ["raw-logs"]
    group_id => "logstash"
    consumer_threads => 8
    auto_offset_reset => "latest"
    decorate_events => true
  }
}

filter {
  # Nginx Access Logs
  if [tag] == "nginx.access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
      overwrite => ["message"]
    }
    geoip {
      source => "clientip"
      target => "geoip"
      database => "/usr/share/GeoIP/GeoLite2-City.mmdb"
    }
    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
    }
    mutate {
      convert => { "bytes" => "integer" }
      remove_field => ["timestamp"]
    }
  }

  # CRM App Logs
  if [tag] == "crm.app" {
    grok {
      match => { "message" => "%{LOGLEVEL:level}: %{GREEDYDATA:log_message}" }
    }
    if [level] == "ERROR" {
      mutate {
        add_tag => ["alert"]
      }
    }
    date {
      match => ["@timestamp", "ISO8601"]
    }
  }

  # EKS Payment Service
  if [tag] == "eks.payment" {
    json {
      source => "message"
      target => "parsed"
    }
    mutate {
      rename => {
        "[parsed][user_id]" => "user_id"
        "[parsed][event]" => "event"
      }
      remove_field => ["parsed"]
    }
    date {
      match => ["[parsed][timestamp]", "ISO8601"]
      target => "@timestamp"
    }
  }

  # Common Enrichments
  mutate {
    add_field => {
      "env" => "prod"
      "region" => "poland"
      "cluster" => "ovh-prod-elk"
    }
  }

  # Drop Low-Value Logs
  if [level] == "DEBUG" and "eks" in [tag] {
    drop {}
  }
}

output {
  if "alert" in [tag] {
    slack {
      url => "${SLACK_WEBHOOK}"
      message => "ELK Alert: %{level} in %{tag}: %{log_message}"
    }
  }
  elasticsearch {
    hosts => ["es-data01.pl.ovh.local:9200", "es-data02.pl.ovh.local:9200", "es-data03.pl.ovh.local:9200"]
    index => "%{tag}-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ES_PASSWORD}"
    ssl => true
    cacert => "/etc/logstash/certs/elastic-ca.pem"
  }
}
```

**Secrets**:
- Slack webhook, ES password from Vault:
  ```bash
  vault kv get secret/elk
  ```
- Ansible sets env vars: `ES_PASSWORD=$(vault read ...)`.

**Granular Flow**:
- **Input**: Logstash pulls a log from `raw-logs` (e.g., Nginx: `192.168.1.1 - - [10/Apr/2025:10:00:00 +0000] "GET /api" 200`).
- **Filter**:
  - Groks Nginx into `clientip`, `status`, `bytes`.
  - GeoIP adds `geoip.city: Warsaw`.
  - Converts `bytes` to integer, sets `@timestamp`.
  - CRM log (`ERROR: User login failed`) gets `level: ERROR`, `alert` tag.
  - EKS JSON (`{"user_id": "456"}`) flattens to `user_id: 456`.
  - Adds `env: prod`, drops EKS debug logs.
- **Output**:
  - Sends to `nginx.access-2025.04.10`, `crm.app-2025.04.10`.
  - Alerts Slack for CRM errors.
- **Failure Mode**: Kafka offsets rewind if Logstash crashes; Elasticsearch retries on 429 errors.
- **Optimization**: 8 consumer threads per server, `queue.type: persisted` for crash recovery.

**Why Logstash?**: Handles complex parsing (grok, geoIP) vs. Fluent Bit’s simplicity. Two servers process ~10TB/day, scaled via Terraform if CPU > 80% (Prometheus).

---

#### 4. Elasticsearch Storage and Indexing
**Role**: Elasticsearch stores, indexes, and searches logs across 10 physical nodes, optimized for volume, speed, and reliability.

**Node Details**:
- **3 Master Nodes** (`es-master01-03.pl.ovh.local`):
  - **Role**: Manage cluster state (mappings, shards), elect leader, handle node joins.
  - **Why 3**: Quorum (2+1) prevents split-brain, survives 1 failure.
  - **Specs**: 8 CPU, 32GB RAM, 500GB SSD (metadata only).
  - **Config**: `node.roles: [master]`, `node.data: false`.
- **5 Data Nodes**:
  - **2 Hot** (`es-data01-02`): Index fresh logs, execute searches.
    - Specs: 16 CPU, 128GB RAM, 4TB NVMe.
    - Config: `node.roles: [data_hot]`.
  - **3 Warm** (`es-data03-05`): Store older logs, lower IOPS.
    - Specs: 12 CPU, 96GB RAM, 8TB HDD.
    - Config: `node.roles: [data_warm]`.
  - **Why 5**: ~10TB/day needs 2 hot for speed (indexing), 3 warm for cost (storage).
- **2 Coordinating Nodes** (`es-coord01-02`):
  - **Role**: Route queries, aggregate results, reduce data node load.
  - **Why 2**: Speeds up Kibana for 100+ users, redundant for uptime.
  - **Specs**: 12 CPU, 64GB RAM, 500GB SSD.
  - **Config**: `node.roles: [coordinating_only]`.

**Cluster Config** (`elasticsearch.yml`):
```yaml
cluster.name: ovh-prod-elk
node.name: ${HOSTNAME}
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["es-master01.pl.ovh.local:9300", "es-master02.pl.ovh.local:9300", "es-master03.pl.ovh.local:9300"]
cluster.initial_master_nodes: ["es-master01", "es-master02", "es-master03"]
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/certs/elastic-certificates.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: /etc/elasticsearch/certs/elastic-certificates.p12
cluster.routing.allocation.disk.threshold_enabled: true
cluster.routing.allocation.disk.watermark.low: 85%
cluster.routing.allocation.disk.watermark.high: 90%
```

**Index Settings**:
- Shards: 5 primaries, 1 replica (10 total).
- Mapping Example (Nginx):
  ```json
  {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "clientip": { "type": "ip" },
        "status": { "type": "integer" },
        "geoip": {
          "properties": {
            "city": { "type": "keyword" }
          }
        }
      }
    }
  }
  ```

**Granular Flow**:
- **Indexing**: Logstash sends a CRM log to `es-data01` (`crm.app-2025.04.10`).
- **Master Role**: `es-master01` assigns 5 primary shards to hot nodes, 5 replicas to others.
- **Hot Phase**: `es-data01` indexes log (`ERROR: User login failed`), stores on NVMe.
- **Search**: Kibana query (`level:ERROR`) hits `es-coord01`, which fetches from `es-data01-02`.
- **Failure Mode**:
  - Master failure: `es-master02` takes over (quorum intact).
  - Data node failure: Replicas on warm nodes serve reads; masters reassign shards.
- **Optimization**:
  - `index.buffer_size: 30%` for indexing speed.
  - `thread_pool.write.size: 32` for concurrent writes.
  - `cluster.routing.allocation.balance.shard: 2` avoids hot spots.

**Why This?**: 3 masters ensure stability, 5 data nodes handle 10TB/day (hot for speed, warm for scale), 2 coordinating nodes cut query latency by ~50% for Kibana.

---

#### 5. Index Lifecycle Management (ILM)
**Role**: Automates log retention and storage tiers for performance, cost, and compliance (90 days).

**Phases**:
- **Hot**: 0-7 days, NVMe, indexing/search-heavy.
- **Warm**: 7-30 days, HDD, frequent queries.
- **Cold**: 30-90 days, HDD, read-only for audits.
- **Delete**: Post-90 days, removes indices.

**ILM Policy**:
```json
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0s",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "7d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "require": { "data": "warm" },
            "number_of_replicas": 1
          },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "require": { "data": "warm" },
            "number_of_replicas": 1
          },
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**Index Template**:
```json
{
  "index_patterns": ["crm.app-*", "nginx.access-*", "eks.payment-*"],
  "settings": {
    "index.lifecycle.name": "log-retention-90d",
    "index.number_of_shards": 5,
    "index.number_of_replicas": 1,
    "index.mapping.total_fields.limit": 1000,
    "index.refresh_interval": "30s"
  }
}
```

**Granular Flow**:
- **Day 1**: `crm.app-2025.04.10` created on `es-data01` (hot), rolls over at 50GB or 7 days.
- **Day 8**: ILM moves to `es-data03` (warm), force-merges to 1 segment for efficiency.
- **Day 31**: Enters cold, read-only, low priority.
- **Day 91**: Deleted, freeing ~100GB.
- **Failure Mode**: If a warm node fails, replicas serve; ILM retries allocation.
- **Optimization**:
  - `max_primary_shard_size: 50gb` caps indexing load.
  - `forcemerge` reduces segments, cuts query time by ~20%.
  - `refresh_interval: 30s` balances indexing vs. search freshness.

**Why ILM?**: Hot NVMe (~$5k/TB) for speed, warm HDD (~$1k/TB) for scale, cold for compliance, delete for GDPR. Saves ~$50k/month vs. all-hot storage.

---

#### 6. Kibana Visualization and Analysis
**Role**: Kibana (1 server, 8 CPU, 32GB RAM) delivers dashboards, searches, and alerts for 100+ users (devs, analysts, SREs).

**Deployment**:
- Server: `kibana.pl.ovh.local`.
- Managed: Ansible (`ansible-playbook kibana.yml`).
- Version: Kibana 8.10.

**Config Details**:
```yaml
server.host: "kibana.pl.ovh.local"
server.port: 5601
server.name: "ovh-prod-kibana"
elasticsearch.hosts: ["https://es-coord01.pl.ovh.local:9200", "https://es-coord02.pl.ovh.local:9200"]
elasticsearch.username: "kibana"
elasticsearch.password: "${KIBANA_PASSWORD}"
xpack.security.enabled: true
xpack.security.authc:
  providers:
    basic:
      enabled: true
xpack.encryptedSavedObjects.encryptionKey: "supersecretkey1234567890abcdef"
xpack.reporting.enabled: true
xpack.reporting.csv.maxSizeBytes: 104857600
```

**Secrets**:
- Password from Vault: `vault kv get secret/kibana`.

**Features**:
- **Searches**: Ad-hoc queries (e.g., `eks.payment-* user_id:456`).
- **Dashboards**:
  - **CRM Errors**: Line chart of `level:ERROR`, ~100 events/hour.
  - **Nginx Traffic**: Histogram of `status:200` vs. `4xx/5xx`.
  - **EKS Latency**: Timeseries of `response_time` from `eks.payment-*`.
- **Alerts**:
  - Rule: `count(level:ERROR) > 50 in 5m` → Slack `#monitoring`.
  - Config:
    ```json
    {
      "name": "CRM Error Spike",
      "index": "crm.app-*",
      "query": "level:ERROR",
      "threshold": 50,
      "time_window": "5m",
      "action": {
        "slack": {
          "webhook": "${SLACK_WEBHOOK}"
        }
      }
    }
    ```
- **Discover**: Raw log exploration with filters (e.g., `geoip.city:Warsaw`).
- **Canvas**: Dynamic reports for execs (e.g., EKS uptime metrics).

**Granular Flow**:
- **Query**: User searches `nginx.access-* status:500` in Discover.
- **Routing**: Kibana sends to `es-coord01`, which queries `es-data01-05`.
- **Response**: Coordinating node aggregates hits (e.g., 100 500s), returns JSON.
- **Render**: Kibana plots a bar chart of errors by `clientip`.
- **Alerting**: If `count(status:500) > 100`, triggers Slack.
- **Failure Mode**: If `es-coord01` fails, `es-coord02` takes over; Kibana retries queries.
- **Optimization**:
  - `query:queryString:options:analyze_wildcard: true` for fast wildcards.
  - `savedObjects:cacheTTL: 60000` caches dashboards.
  - `xpack.reporting.queue.timeout: 120000` for long CSV exports.

**Why Kibana?**: Intuitive for non-tech users, scales for 100+ concurrent sessions, integrates with Elasticsearch for sub-second queries.

---

#### 7. Observability and Security Integrations
**Prometheus/Grafana**:
- **Scrapers**:
  - `elasticsearch-exporter`: `elasticsearch_indices_docs_count`, `elasticsearch_jvm_memory_used_bytes`.
  - `kafka-exporter`: `kafka_topic_partition_current_offset`.
  - `fluentbit_exporter`: `fluentbit_input_bytes_total`.
- **Alerts**:
  - `elasticsearch_cluster_health_status != green` → PagerDuty.
  - `logstash_pipeline_events_out_total < 1000 for 5m` → Slack.
- **Dashboard**:
  ```yaml
  panels:
  - title: "Indexing Rate"
    type: graph
    query: rate(elasticsearch_indices_indexing_index_total[5m])
  ```

**OpenTelemetry/Jaeger**:
- **EKS Traces**: Payment Service emits `trace_id: abc123`.
- **Correlation**: Kibana links traces to logs via `trace_id` field.
- **Config**:
  ```yaml
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: "jaeger.pl.ovh.local:4317"
  exporters:
    elasticsearch:
      endpoints: ["https://es-data01.pl.ovh.local:9200"]
  ```

**Vault**:
- **Secrets**: Stores `elastic`, `kibana` passwords.
- **Access**: Logstash pulls via `VAULT_TOKEN`:
  ```bash
  vault kv get secret/elk
  ```
- **Rotation**: Monthly via Ansible cron (`ansible-playbook rotate-secrets.yml`).

**Granular Flow**:
- **Monitoring**: Prometheus scrapes `es-data01` every 15s, plots JVM heap in Grafana.
- **Tracing**: Jaeger trace (`abc123`) links to `eks.payment-*` log in Kibana.
- **Security**: Vault injects `ES_PASSWORD` into Logstash, audited via `vault audit`.
- **Failure Mode**: Prometheus alerts on missing metrics; Vault fallback to cached secrets.
- **Optimization**:
  - `scrape_interval: 15s` balances load vs. granularity.
  - Jaeger sampling at 0.1% for high-traffic EKS apps.

---

### End-to-End Example: CRM Error Log
1. **Source**: CRM VM writes `ERROR: User login failed - ID: 123` to `/var/log/crm/app.log`.
2. **Fluent Bit**:
   - Tails log, tags `crm.app`, adds `hostname: vm-crm01`, `env: prod`.
   - Sends to `raw-logs` partition 7.
3. **Kafka**:
   - Stores in `kafka01`, replicates to `kafka02`.
   - Commits offset 123456.
4. **Logstash**:
   - Consumes offset 123456, groks to `level: ERROR`, `log_message: User login failed`.
   - Adds `alert` tag, sends to `crm.app-2025.04.10`.
   - Posts to Slack: “ERROR in crm.app: User login failed”.
5. **Elasticsearch**:
   - `es-master01` assigns shard 1 to `es-data01` (hot).
   - Indexes log with `@timestamp: 2025-04-10T10:00:00`.
   - ILM moves to `es-data03` (warm) on day 8, cold on day 31, deletes on day 91.
6. **Kibana**:
   - User queries `crm.app-* level:ERROR`.
   - `es-coord01` fetches from `es-data01`, renders 10 errors in Discover.
   - Dashboard plots error count, alerts if >50 in 5m.
7. **Observability**:
   - Prometheus tracks `elasticsearch_indices_docs_count`.
   - Jaeger links to trace if CRM API failed.

**Metrics**:
- Latency: ~500ms from Fluent Bit to Kibana.
- Throughput: ~100k events/sec peak.
- Storage: ~100GB/index/day, ~9TB total at 90 days.

---

### Operational Nuances

#### Scaling
- **Fluent Bit**: Add pods via Argo CD (`kubectl scale ds/fluent-bit --replicas=50`).
- **Kafka**: Increase partitions:
  ```bash
  kafka-topics.sh --alter --topic raw-logs --partitions 32
  ```
- **Logstash**: Terraform adds servers:
  ```hcl
  resource "vsphere_host" "logstash" {
    count      = var.logstash_nodes
    hostname   = "logstash${count.index + 1}.pl.ovh.local"
    memory     = 64000
    cpu        = 16
  }
  ```
- **Elasticsearch**: Add data nodes:
  ```hcl
  resource "vsphere_host" "es_data" {
    count      = var.data_nodes
    hostname   = "es-data${count.index + 1}.pl.ovh.local"
    memory     = 128000
    cpu        = 16
  }
  ```

#### Failure Handling
- **Fluent Bit**: Disk buffer (`/fluent-bit/storage/`) if Kafka’s down.
- **Kafka**: Broker failure triggers replication; Logstash rewinds offsets.
- **Logstash**: Persisted queue (`queue.type: persisted`) survives crashes.
- **Elasticsearch**: Master failover, shard reallocation; replicas serve reads.
- **Kibana**: Retries queries, caches dashboards.

#### Performance Tuning
- **Fluent Bit**: `Mem_Buf_Limit: 5MB` caps memory, `Flush: 1` for EKS speed.
- **Kafka**: `num.io.threads: 16` for disk I/O, `compression.type: gzip` saves bandwidth.
- **Logstash**: `pipeline.batch.size: 1000` reduces network calls.
- **Elasticsearch**: `index.translog.flush_threshold_size: 512mb` for write efficiency.
- **Kibana**: `elasticsearch.requestTimeout: 60000` for long queries.

#### Security
- **TLS**: Elasticsearch uses `elastic-certificates.p12`, Logstash/Kibana use CA-signed certs.
- **Auth**: Role-based access (`elastic` for admin, `kibana_user` for viewers).
- **Network**: OVHcloud firewall restricts to `10.0.0.0/16`, Direct Connect to AWS.

#### Backup and Recovery
- **Snapshots**:
  ```json
  PUT _snapshot/elk_backup
  {
    "type": "fs",
    "settings": {
      "location": "/mnt/backups/elk",
      "compress": true
    }
  }
  POST _snapshot/elk_backup/daily_2025-04-10
  {
    "indices": "crm.app-*,nginx.access-*",
    "ignore_unavailable": true
  }
  ```
- **Restore**:
  ```json
  POST _snapshot/elk_backup/daily_2025-04-10/_restore
  {
    "indices": "crm.app-2025.04.10"
  }
  ```
- **Schedule**: Nightly via cron, stored in OVHcloud’s S3-compatible storage.

---

### Failure Modes and Mitigations
1. **Fluent Bit Overload**:
   - Symptom: High CPU, dropped logs.
   - Fix: Scale pods/VMs, increase `Mem_Buf_Limit`.
   - Monitor: `fluentbit_output_dropped_records_total`.
2. **Kafka Lag**:
   - Symptom: Logstash backlog, `kafka_consumer_lag > 10000`.
   - Fix: Add partitions, scale Logstash threads.
   - Alert: Grafana notifies at 10k lag.
3. **Logstash Parsing Error**:
   - Symptom: `_grokparsefailure` tags in Elasticsearch.
   - Fix: Debug grok patterns, test with `stdin` input.
   - Log: `/var/log/logstash/logstash-plain.log`.
4. **Elasticsearch Red Cluster**:
   - Symptom: Unassigned shards, `curl _cluster/health` shows `red`.
   - Fix: Reallocate shards (`POST _cluster/reroute`), add nodes.
   - Alert: Prometheus triggers on `status != green`.
5. **Kibana Timeout**:
   - Symptom: “Request timed out” for complex queries.
   - Fix: Increase `elasticsearch.requestTimeout`, optimize query.
   - Cache: `savedObjects:cacheTTL: 120000`.

---

### Interview-Ready Deep Dive
“Our ELK Stack in OVHcloud’s Poland data center handles 10TB of logs daily from CRM, ERP, EKS microservices, Nginx, and MySQL. Fluent Bit tails logs on VMs and EKS pods, tagging them (e.g., `crm.app`), enriching with metadata (`env: prod`), and shipping to Kafka’s `raw-logs` topic with 16 partitions for buffering. Logstash consumes from Kafka, groks Nginx logs, parses JSON from EKS, adds geoIP, and drops debug noise, indexing to Elasticsearch as `nginx.access-2025.04.10`. Elasticsearch runs on 10 nodes: 3 masters for quorum, 2 hot data nodes for indexing, 3 warm for storage, and 2 coordinating for query speed, handling ~100k events/sec. ILM shifts logs from hot NVMe (7 days) to warm HDD (30 days), cold read-only (90 days), then deletes for GDPR. Kibana serves 100+ users with dashboards for errors, latency, and traffic, alerting Slack on spikes. Prometheus tracks indexing rates, Jaeger links traces, and Vault secures credentials. I tune configs, scale with Terraform, and debug failures, ensuring observability across our hybrid cloud.”

---

This is as granular as it gets, covering every byte from source to screen. If you want *even more* depth on a specific part—say, Logstash grok patterns, Elasticsearch shard math, or Kibana query optimization—point me there, and I’ll go deeper. How’s this for your interview prep? Ready to nail it?
