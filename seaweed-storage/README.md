# 📦 SeaweedFS File Storage Architecture  
A practical deployment of SeaweedFS used as a **file saver** with strict separation between **internal write access** and **public read‑only access**.

---

## Preparing Storage Directory

```bash
mkdir data
sudo chown -R 1000:1000 data
sudo chmod -R 755 data
```

---

## 🧩 Architecture Overview

| Component | Port | Access | Purpose |
|----------|------|--------|---------|
| **Master** | 9333 | Internal | Coordinates volumes & metadata |
| **Filer (Write)** | 9555 | Internal | Handles uploads & metadata writes |
| **Filer (Read‑Only)** | 8888 | Public | Safe public download endpoint |
| **Volume Servers** | 8081 / 8082 | Internal | Store actual file data |

## Quick Reference: Curl Commands

| Operation | Port | Command | Status |
|-----------|------|---------|--------|
| **Upload** | 9555 | `curl -F "file=@path/file"` | ✅ Works |
| **Download** | 9555 | `curl http://localhost:9555/public/file` | ✅ Works |
| **Download (Public)** | 8888 | `curl http://localhost:8888/public/file` | ✅ Works |
| **Upload (Public)** | 8888 | `curl -F "file=@path/file"` | ❌ 405 Denied |

---

## 🖼️ Architecture Diagram

```
┌──────────────────┐
│   Master Server  │ Cluster topology, volume mapping, rack awareness
└────────┬─────────┘
         │
    ┌────┴─────────┐
    │              │
    ▼              ▼
┌───────┐      ┌───────┐
│ Filer │      │ Filer │ POSIX API, HTTP endpoints, hierarchy
└───┬───┘      └──┬────┘
    │             │
    ├──────┬──────┼──────┐
    │      │      │      │
    ▼      ▼      ▼      ▼
  ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │ V1 │ │ V2 │ │ V3 │ │ V4 │ Volume servers (one per disk/storage)
  └────┘ └────┘ └────┘ └────┘
    │      │      │      │
  Disk1  Disk2  Disk3  Disk4
```
**Key principle**: Each volume server is **bound to one storage device**. Multiple disks require multiple volume server instances.


---

## 🐳 Docker Compose Setup  

---

### 🟦 Master Server (9333)

```yaml
master:
  image: chrislusf/seaweedfs:4.31
  user: seaweed
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
  networks:
    - ngelmak-net
```

#### Key parameters

| Parameter | Purpose | Default |
|-----------|---------|---------|
| `-ip=master` | Service hostname (Docker network) | — |
| `-port=9333` | Master RPC port | `9333` |
| `-mdir=/data` | Metadata persistence directory | `./data` |
| `-defaultReplication=000` | Default rule: `XYZ` (same-rack, same-DC, cross-DC) | `000` |
| `-volumeSizeLimitMB=1024` | Max volume size; auto-creates new ones when full | `30720` (30 GB) |
| `-peers=none` | For multi-master clusters; single node uses `none` | `none` |

---

### 🟩 Filer — Write Access (9555)

Exposes a **POSIX-like API** (directory structure) and **HTTP endpoints** for file operations.

```yaml
filer:
  image: chrislusf/seaweedfs:4.31
  user: seaweed
  container_name: seaweed-filer
  command: >
    filer
    -master=master:9333
    -ip=filer
    -port=9555
    -defaultReplicaPlacement=000
  depends_on:
    - master
    - volume1
  ports:
    - "9555:9555"
  volumes:
    - ./data/filer_meta:/data/filer_meta
    - ./data/config/filer.toml:/etc/seaweedfs/filer.toml:ro
  networks:
    - ngelmak-net
```

#### Key parameters  
| Parameter | Purpose | Default |
|-----------|---------|---------|
| `-master=master:9333` | Master server endpoint | — |
| `-ip=filer` | Service hostname (Docker network) | — |
| `-port=9555` | HTTP API port | `9555` |
| `-defaultReplicaPlacement=000` | Replica distribution rule | `000` |

**Metadata backends**: Default is LevelDB; can swap for MySQL, PostgreSQL, or Redis via `-metaFolder` configuration.

#### Filer Configuration (`filer.toml`)

```toml
[leveldb2]
enabled = true
dir = "/data/filer_meta"

[http]
disableDirListing = true

[http.access]
read = ["/public/*"]
write = ["/public/*"]
```

This file **overrides** CLI flags and ensures:  
- Only `/public/*` is accessible  
- Directory listing is disabled  
- Metadata stored in `/data/filer_meta`  


---

### 🟧 Filer — Read‑Only (8888)

```yaml
filer-ro:
  image: chrislusf/seaweedfs:4.31
  user: seaweed
  container_name: seaweed-filer-read
  command: >
    filer
    -master=master:9333
    -ip=filer-ro
    -port=9555
    -port.readonly=8888
    -disableDirListing=true
    -exposeDirectoryData=false
    -ui.deleteDir=false
  depends_on:
    - master
    - volume1
    - filer
  ports:
    - "8888:8888"
  volumes:
    - ./data/config/filer.toml:/etc/seaweedfs/filer.toml:ro
  networks:
    - ngelmak-net
```

#### Key parameters  
| Parameter | Purpose | Default |
|-----------|---------|---------|
| `-port.readonly=8888` | Public read-only HTTP endpoint (separate from main filer port) | `8888` |
| `-disableDirListing=true` | Disables directory browsing; clients cannot list folder contents | `false` |
| `-exposeDirectoryData=false` | Hides folder structure from API responses; restricts metadata exposure | `true` |
| `-ui.deleteDir=false` | Disables folder deletion via web UI; prevents accidental recursive deletes | `true` |

---

### 🟫 Volume Server 1 (8081)

**Store actual file data**. One container per physical disk or storage device.

```yaml
volume1:
  image: chrislusf/seaweedfs:4.31
  user: seaweed
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
  networks:
    - ngelmak-net
```

#### Key parameters  
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

### 🟫 Volume Server 2 (8082)

```yaml
volume2:
  image: chrislusf/seaweedfs:4.31
  user: seaweed
  container_name: seaweed-volume2
  command: >
    volume
    -master=master:9333
    -ip=volume2
    -ip.bind=volume2
    -port=8080
    -dir=/data
    -max=2
  depends_on:
    master:
      condition: service_healthy
  ports:
    - "8082:8080"
  volumes:
    - ./data/volume2:/data
  networks:
    - ngelmak-net
```


---

## 🔍 Start Verification & Monitoring


### 🖥️ Server 1 (Master + 2x Filer + Volume)

```bash
docker compose -f docker-compose.seaweed.yml up -d
```

### 🖥️ Server 2+ (Volume Servers)

```bash
docker compose -f docker-compose.volume2.yml up -d
```

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
| `http://filer:9555/?pretty=y` | GET | Health check & cluster info |
| `http://filer:8888/path/to/file` | GET | Retrieve file |
| `http://filer:9555/path/to/file` | PUT | Upload file |
| `http://filer:9555/path/to/file` | DELETE | Delete file |
| `http://filer:9555/admin/dir/status?path=/uploads` | GET | Directory statistics |


---

## 🧪 Curl Testing (With Expected Responses)

---

### Upload (Write Port 9555)

```bash
curl -i -F "file=@examples/image1.jpg" \
  "http://localhost:9555/public/image1.jpg"
```

**Expected response:**

```
HTTP/1.1 201 Created
Content-Md5: <checksum>
{"name":"image1.jpg","size":3135682}
```

---

### Download (Public Read‑Only Port 8888)

```bash
curl -o image1.jpg "http://localhost:8888/public/image1.jpg"
```

**Expected response:**

```
100 3135k  100 3135k    0     0  12.5M      0 --:-- --:-- --:-- 12.5M
```

---

### Upload to Read‑Only Port (Should Fail)

```bash
curl -i -F "file=@examples/image2.jpg" \
  "http://localhost:8888/public/image2.jpg"
```

**Expected response:**

```
HTTP/1.1 405 Method Not Allowed
Content-Length: 0
```

---

## 🛡️ Security Practices

| Practice | Status |
|---------|--------|
| Separate read/write ports | ✔️ |
| Only expose read‑only port | ✔️ |
| Directory listing disabled | ✔️ |
| Folder structure hidden | ✔️ |
| UI delete disabled | ✔️ |
| Internal Docker network | ✔️ |