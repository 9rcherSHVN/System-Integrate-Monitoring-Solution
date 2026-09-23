## Phase 1: Spool/Replay Design

### Purpose

**Spool/replay is a store-and-forward reliability mechanism** for custom collectors.

A collector may successfully gather certificate, disk, or ERP folder data but be temporarily unable to send it to Azure Monitor because of:

- Internet/proxy outage.
- Azure Monitor/DCE endpoint temporary unavailability.
- DNS or TLS issue.
- Temporary Microsoft Entra/Azure Arc token issue.
- Transient HTTP failure or throttling.
- Short-lived DCR authorization propagation delay.

Without a spool, the collector would either:

1. Discard the records after a failed API call, creating a historical monitoring gap; or  
2. Keep retrying indefinitely, potentially blocking the next scheduled task and consuming resources.

The spool design preserves the collected, sanitized JSON payload locally and retries it later.

```text
Scheduled collector run
    │
    ├── Collect certificate/disk/folder metrics
    ├── Build JSON payload
    ├── Attempt Logs Ingestion API submission
    │      │
    │      ├── Success
    │      │     └── Data reaches Log Analytics
    │      │
    │      └── Retryable failure after retry limit
    │             └── Save payload to local spool directory
    │
    ▼
Separate scheduled spool-replay task
    │
    ├── Reads oldest queued payload
    ├── Obtains fresh Azure Arc managed-identity token
    ├── Retries Azure Monitor submission
    ├── Deletes spool file only after successful submission
    └── Leaves it queued if submission still fails
```

---

### What Is Stored in the Spool

A spool file is a local JSON document containing:

- Collector name.
- Creation timestamp.
- DCE or DCR ingestion endpoint.
- DCR immutable ID.
- Stream name.
- Already serialized telemetry payload.

Example conceptual spool record:

```json name=CertificateInventory-spool-example.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "CertificateInventory",
  "CreatedUtc": "2026-09-23T02:15:31.0000000Z",
  "IngestionEndpoint": "https://<dce-ingestion-endpoint>",
  "DcrImmutableId": "dcr-<certificate-dcr-immutable-id>",
  "StreamName": "Custom-ServerCertificateInventoryRaw",
  "Payload": "[{\"TimeGenerated\":\"2026-09-23T02:15:00Z\",\"Computer\":\"ERP-MW-01\",\"Subject\":\"CN=api.example.com\"}]"
}
```

The payload should contain only the same approved telemetry that would otherwise be sent to Log Analytics.

It must **not** contain:

- Access tokens.
- HIMDS challenge secrets.
- Passwords.
- Private certificate keys.
- PFX files.
- Workspace shared keys.
- Service-principal secrets.
- Unapproved ERP business data.

---

### Local Spool Directory Layout

Each collector receives its own queue:

```text
C:\ProgramData\Contoso\AzureMonitorCollectors\
└── Spool\
    ├── CertificateInventory\
    │   ├── CertificateInventory-20260923021531001-<guid>.json
    │   └── ...
    ├── ErpFolderMetrics\
    │   ├── ErpFolderMetrics-20260923030011004-<guid>.json
    │   └── ...
    └── WindowsDiskMetrics\
        ├── WindowsDiskMetrics-20260923031508017-<guid>.json
        └── ...
```

Separate directories are useful because:

- A certificate-ingestion issue does not block disk replay.
- Spool size can be attributed to one collector.
- Different collection frequencies have different queue growth patterns.
- Operations can investigate and recover one stream safely.

---

### Execution Sequence

#### Step 1 — New Collection Is Attempted

The collector creates one or more JSON batches and calls the Logs Ingestion API.

```text
Collect → batch → get Arc identity token → HTTPS POST
```

The collector retries transient failures a limited number of times, for example:

```text
Attempt 1: immediately
Attempt 2: wait 2 seconds
Attempt 3: wait 4 seconds
Attempt 4: wait 8 seconds
```

This is exponential backoff.

#### Step 2 — Failure Is Classified

Typical retryable failure conditions include:

| Condition | Typical response |
|---|---|
| Temporary endpoint timeout | Retry, then spool |
| Proxy/network interruption | Retry, then spool |
| HTTP 429 throttling | Retry with backoff, then spool |
| HTTP 500/502/503/504 | Retry, then spool |
| Temporary Entra/Azure Arc token failure | Retry, then spool |

Typical non-retryable configuration failures include:

| Condition | Typical response |
|---|---|
| HTTP 400 | Payload does not match DCR stream schema; investigate configuration/code |
| HTTP 401 | Token audience/token issue; investigate Arc identity/time/network |
| HTTP 403 | DCR RBAC issue; investigate role assignment |
| HTTP 404 | Wrong endpoint, DCR immutable ID, stream, or API path |
| Invalid JSON | Collector defect or schema mismatch |

A production implementation can classify HTTP status codes more precisely. The pilot implementation should at least distinguish transient endpoint/network failures from configuration/schema failures.

#### Step 3 — Batch Is Spool-Saved

If all retry attempts fail and local spooling is enabled:

```text
Payload JSON
    → write temporary spool file
    → atomically rename to final .json
```

Use an atomic write pattern to prevent spool-replay from reading a partially written file:

```text
Write: file.json.tmp
    ↓
Flush/close file
    ↓
Rename: file.json.tmp → file.json
```

This should be added to the production-hardening version of `Save-CollectorSpoolBatch`.

#### Step 4 — Scheduled Replay Runs

The replay task runs independently, typically every 15 minutes.

```text
Contoso-Monitor-<CollectorName>-SpoolReplay
```

It:

1. Reads queued `.json` files in oldest-first order.
2. Requests a new token from Azure Arc HIMDS.
3. Posts the saved payload using its saved endpoint/DCR/stream values.
4. Deletes the spool file only after Azure Monitor accepts the request.
5. Leaves the file in place if replay still fails.

The collector does not reuse a saved access token because Azure access tokens expire and should never be written to disk.

---

### Why a Separate Replay Task Is Preferable

A separate replay task avoids placing all recovery responsibility on the next main collector run.

For example:

```text
Certificate collector: daily at 02:15
Disk collector: every 15 minutes
```

If the daily certificate collector fails at 02:15 because Azure connectivity is unavailable, waiting until the next day to replay the certificate inventory is undesirable.

A separate replay task running every 15 minutes means it can submit the queued certificate data as soon as connectivity returns.

---

### Spool Limits and Controls

Spooling must be bounded. It is not an unlimited offline data warehouse.

Recommended controls:

| Control | Initial pilot recommendation |
|---|---|
| Spool retention | 3 days |
| Replay frequency | Every 15 minutes |
| Maximum spool file size | Below Logs Ingestion API batch limit; target below 900 KB |
| File permissions | Collector account: Modify; support: Read; administrators: Full Control |
| Oldest-first replay | Yes |
| Delete after success | Yes, immediately after accepted submission |
| Alert threshold | Warning at 10 queued files; critical at 100 or oldest file older than 60 minutes |
| Log retention | 30 days |
| Spool disk location | Local system/app disk with capacity monitoring |
| Secrets in spool | Prohibited |

### Important Operational Trade-Off

When telemetry is replayed after a network outage, the record may be ingested later than it was collected. Preserve the original `TimeGenerated` timestamp in the payload. This allows KQL to reflect when the event/metric was actually collected, rather than when Azure received it.

---

