# 🎯 SeaweedFS Cluster (Master + Filer + Volume Servers)

A **high‑performance, distributed object storage system** built with **SeaweedFS**, optimized for storing large numbers of small to medium files with minimal overhead.

## 🤔 Why SeaweedFS?

- **Horizontal scalability**: Add volume servers without cluster downtime
- **Efficient small-file storage**: Optimized for millions of files (photos, documents, metadata)
- **POSIX-compatible API**: Familiar directory structure and HTTP endpoints
- **Low operational overhead**: Simple topology management and automatic volume allocation
- **Flexible replication**: Control redundancy at cluster or per-file level

---

## 📚 Theoretical Foundation

SeaweedFS implements a **master-slave distributed architecture** with the following key concepts:

### 🔹 Volume-Based Storage Model

Unlike traditional distributed filesystems (HDFS, CephFS) that stripe data across nodes, SeaweedFS uses **volumes** as atomic storage units:

- Each **volume** is a fixed-size file (typically 1–30 GB) containing serialized file metadata and content
- Volumes are immutable once closed, enabling efficient sequential I/O and replication
- The **master** tracks volume-to-node mappings; the **filer** translates hierarchical paths to volume locations

**Benefit**: Minimal metadata overhead; efficient for write-once/append-rarely workloads.

### 🔹 Two-Tier Metadata Management

1. **Master tier**: Cluster topology, volume assignments, rack/DC awareness (in-memory + persistent log)
2. **Filer tier**: Directory hierarchy, file-to-volume mappings (LevelDB/relational backend)

This decoupling allows independent scaling of storage and namespace layers.

### 🔹 Placement Rules & Replication

Replication placement is controlled by a **three-digit rule** `XYZ`:
- **X**: Extra replicas in the same rack
- **Y**: Extra replicas in the same data center
- **Z**: Extra replicas in different data centers

Example: `101` = 1 copy locally, 1 copy elsewhere in DC, 1 copy in different DC (3 replicas total).

---

## 🏗️ Architecture

```
┌──────────────────┐
│   Master Server  │◄─ Cluster topology, volume mapping, rack awareness
└────────┬─────────┘
         │
    ┌────┴─────────┐
    │              │
    ▼              ▼
┌───────┐      ┌───────┐
│ Filer │      │ Filer │  ◄─ POSIX API, HTTP endpoints, hierarchy
└───┬───┘      └──┬────┘
    │             │
    ├──────┬──────┼──────┐
    │      │      │      │
    ▼      ▼      ▼      ▼
  ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │ V1 │ │ V2 │ │ V3 │ │ V4 │  ◄─ Volume servers (one per disk/storage)
  └────┘ └────┘ └────┘ └────┘
    │      │      │      │
  Disk1  Disk2  Disk3  Disk4
```

**Key principle**: Each volume server is **bound to one storage device**. Multiple disks require multiple volume server instances.

---

## ⚡ Quick Start

### 🖥️ Server 1 (Master + Filer)

```bash
docker compose -f docker-compose.master-filer.yml up -d
```

### 🖥️ Server 2+ (Volume Servers)

```bash
docker compose -f docker-compose.volume.yml up -d
```

Then verify:

```bash
docker exec seaweed-filer weed shell
> volume.list
> master.volumeStats
```

---

## ⚙️ Services Configuration

### 🟦 Master Server

Manages **cluster topology**, **volume allocation**, and **replication state**.

```yaml
master:
  image: chrislusf/seaweedfs:4.31
  container_name: seaweed-master
  command: >
    master
    -ip=master
    -port=9333
    -mdir=/data
    -peers=none
    -defaultReplication=000
    -volumeSizeLimitMB=1024
  ports:
    - "9333:9333"
  volumes:
    - ./data/master:/data
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9333/cluster/status"]
    interval: 10s
    timeout: 5s
    retries: 5
  restart: on-failure:5
```

| Parameter | Purpose | Default |
|-----------|---------|---------|
| `-ip=master` | Service hostname (Docker network) | — |
| `-port=9333` | Master RPC port | `9333` |
| `-mdir=/data` | Metadata persistence directory | `./data` |
| `-defaultReplication=000` | Default rule: `XYZ` (same-rack, same-DC, cross-DC) | `000` |
| `-volumeSizeLimitMB=1024` | Max volume size; auto-creates new ones when full | `30720` (30 GB) |
| `-peers=none` | For multi-master clusters; single node uses `none` | `none` |

---

### 🟩 Filer Server

Exposes a **POSIX-like API** (directory structure) and **HTTP endpoints** for file operations.

```yaml
filer:
  image: chrislusf/seaweedfs:4.31
  container_name: seaweed-filer
  command: >
    filer
    -master=master:9333
    -ip=filer
    -port=8888
    -defaultReplicaPlacement=000
    -defaultStoreDir=/data/filer_store
  depends_on:
    master:
      condition: service_healthy
  ports:
    - "8888:8888"
  volumes:
    - ./data/filer:/data
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8888/?pretty=y"]
    interval: 10s
    timeout: 5s
    retries: 5
  restart: on-failure:5
```

| Parameter | Purpose | Default |
|-----------|---------|---------|
| `-master=master:9333` | Master server endpoint | — |
| `-ip=filer` | Service hostname (Docker network) | — |
| `-port=8888` | HTTP API port | `8888` |
| `-defaultReplicaPlacement=000` | Replica distribution rule | `000` |
| `-defaultStoreDir=/data/filer_store` | Metadata storage (LevelDB) | `./filer_store` |

**Metadata backends**: Default is LevelDB; can swap for MySQL, PostgreSQL, or Redis via `-metaFolder` configuration.

---

### 🟧 Volume Servers

**Store actual file data**. One container per physical disk or storage device.

```yaml
volume1:
  image: chrislusf/seaweedfs:4.31
  container_name: seaweed-volume1
  command: >
    volume
    -master=master:9333
    -ip=volume1
    -ip.bind=volume1
    -port=8080
    -dir=/data
    -max=2
  depends_on:
    master:
      condition: service_healthy
  ports:
    - "8081:8080"
  volumes:
    - ./data/volume1:/data
```

| Parameter | Purpose | Default |
|-----------|---------|---------|
| `-master=master:9333` | Master server endpoint | — |
| `-ip=volume1` | Advertised hostname | — |
| `-ip.bind=volume1` | Bind address | `0.0.0.0` |
| `-port=8080` | Service port | `8080` |
| `-dir=/data` | Volume storage directory (one disk) | — |
| `-max=2` | Max volumes this server can host | `8` |
| `-rack=rack1` | Rack identifier (optional, for placement) | default |
| `-dataCenter=dc1` | Data center identifier (optional, for geo-placement) | default |

---

## 📦 Deployment for Multiple Servers

### 🖥️ **Server 1** (`docker-compose.master-filer.yml`)

Contains master and filer services only.

```yaml
version: "3.9"

services:
  master:
    image: chrislusf/seaweedfs:4.31
    container_name: seaweed-master
    command: >
      master
      -ip=0.0.0.0
      -port=9333
      -mdir=/data
      -peers=none
      -defaultReplication=000
      -volumeSizeLimitMB=1024
    ports:
      - "9333:9333"
    volumes:
      - ./data/master:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9333/cluster/status"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: on-failure:5

  filer:
    image: chrislusf/seaweedfs:4.31
    container_name: seaweed-filer
    command: >
      filer
      -master=master:9333
      -ip=0.0.0.0
      -port=8888
      -defaultReplicaPlacement=000
      -defaultStoreDir=/data/filer_store
    depends_on:
      master:
        condition: service_healthy
    ports:
      - "8888:8888"
    volumes:
      - ./data/filer:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888/?pretty=y"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: on-failure:5
```

**Start**:
```bash
docker compose -f docker-compose.master-filer.yml up -d
```

---

### 🖥️ **Server 2+** (`docker-compose.volume.yml`)

Contains volume servers. Deploy on each storage node.

```yaml
version: "3.9"

services:
  volume1:
    image: chrislusf/seaweedfs:4.31
    container_name: seaweed-volume1
    command: >
      volume
      -master=<MASTER_IP>:9333
      -ip=0.0.0.0
      -port=8080
      -dir=/data
      -max=2
      -rack=rack1
    ports:
      - "8081:8080"
    volumes:
      - ./data/volume1:/data
    restart: on-failure:5

  volume2:
    image: chrislusf/seaweedfs:4.31
    container_name: seaweed-volume2
    command: >
      volume
      -master=<MASTER_IP>:9333
      -ip=0.0.0.0
      -port=8080
      -dir=/data2
      -max=2
      -rack=rack1
    ports:
      - "8082:8080"
    volumes:
      - ./data/volume2:/data2
    restart: on-failure:5
```

Replace `<MASTER_IP>` with the actual IP or hostname of Server 1.

**Start**:
```bash
docker compose -f docker-compose.volume.yml up -d
```

---

## 📈 Scaling: Add More Volume Servers

On **Server 3, 4, 5...**, modify `docker-compose.volume.yml` with unique container names and ports:

```yaml
volume3:
  image: chrislusf/seaweedfs:4.31
  container_name: seaweed-volume3
  command: >
    volume
    -master=<MASTER_IP>:9333
    -ip=0.0.0.0
    -port=8080
    -dir=/data
    -max=2
    -rack=rack2
  ports:
    - "8083:8080"
  volumes:
    - ./data/volume3:/data
  restart: on-failure:5
```

The master **automatically detects** new volume servers and begins allocating volumes.

---

## 💾 Storage Design Deep Dive

### Volume Lifecycle

A **volume** is a write-once, immutable container:

1. **Creation**: Master allocates volume ID when first file is written
2. **Growth**: Files accumulate until volume reaches `-volumeSizeLimitMB` limit
3. **Closure**: Automatically closed; no new writes allowed
4. **Replication**: Closed volumes are replicated per placement rule

**Example** with `-volumeSizeLimitMB=1024` and `-max=2`:

| State | Volume 1 | Volume 2 | Volume 3 |
|-------|----------|----------|----------|
| Growing | 0–1 GB | — | — |
| Growing | 1 GB (closed) | 0–500 MB | — |
| Growing | 1 GB (closed) | 1 GB (closed) | — |
| **Blocked** | 1 GB (closed) | 1 GB (closed) | *Cannot create (max=2)* |

When volume capacity is exhausted, uploads fail unless:
- `-max` is increased
- Old volumes are deleted or archived
- New volume servers are added

### Replication Guarantees

With `defaultReplication=001`:
- Each file has **2 copies minimum** (original + 1 cross-datacenter)
- If a volume server fails, master redirects reads to replicas
- Replication is **automatic and asynchronous** after volume closure

---

## 🔍 Verification & Monitoring

### 📊 Check cluster health

```bash
# Access master dashboard
curl http://<MASTER_IP>:9333/cluster/status

# Access filer shell
docker exec seaweed-filer weed shell

# Inside shell:
> volume.list              # All volumes, replicas, free slots
> master.volumeStats       # Aggregate statistics
> fs.ls /                  # List root directory
> fs.du /uploads           # Disk usage per directory
```

### 🌐 Filer API endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `http://filer:8888/?pretty=y` | GET | Health check & cluster info |
| `http://filer:8888/path/to/file` | GET | Retrieve file |
| `http://filer:8888/path/to/file` | PUT | Upload file |
| `http://filer:8888/path/to/file` | DELETE | Delete file |
| `http://filer:8888/admin/dir/status?path=/uploads` | GET | Directory statistics |

### 📈 Prometheus metrics

Master exposes metrics on port `9333`:
```bash
curl http://<MASTER_IP>:9333/metrics
```

---

## 🏭 Production Considerations

| Aspect | Recommendation |
|--------|-----------------|
| **Replication** | Set `-defaultReplication` to `001` or higher; geo-distribute volume servers |
| **Volume limits** | Monitor `-max` values; alert before exhaustion |
| **Metadata backup** | Persist master and filer data to reliable storage (NAS, object store) |
| **Multi-master HA** | Use `-peers` for master clustering; configure load balancer for filers |
| **Network isolation** | Use overlay networks (Swarm/K8s) or secure VPNs between nodes |
| **Port exposure** | Place filer behind reverse proxy (nginx/HAProxy); restrict master access |
| **Monitoring** | Export metrics to Prometheus; alert on volume fullness, server down, replication lag |

---

## 🎛️ Configuration Reference

| Component | Key Customizations |
|-----------|-------------------|
| **Master** | replication policy (`-defaultReplication`), volume size (`-volumeSizeLimitMB`), RPC port, metadata directory, multi-master peers |
| **Filer** | replica placement (`-defaultReplicaPlacement`), HTTP port, metadata backend (LevelDB/MySQL/Postgres/Redis) |
| **Volume** | disk path (`-dir`), max volumes (`-max`), RPC port, rack/datacenter labels (`-rack`, `-dataCenter`) |

Modify these values in the compose file `command` sections **before deployment**. Changes to running services require restart.