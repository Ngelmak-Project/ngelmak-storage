# 📘 Ngelmak Postgres Service

This document explains how to manage the **Ngelmak PostgreSQL service** used by other microservices in the system.  

---
  
# 🚀 1. Service Overview

The PostgreSQL instance is defined in `docker-compose.yml`:

```yaml
services:
  ngelmak-pgdb:
    image: postgres:16
    container_name: ngelmak-pgdb
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: changeme
      POSTGRES_DB: postgres
    ports:
      - "5432:5432"
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    networks:
      - ngelmak-net

networks:
  ngelmak-net:
    external: true
```

### Default credentials

| Key | Value |
|-----|--------|
| Host | `localhost` |
| Port | `5432` |
| Admin user | `admin` |
| Admin password | `changeme` |
| Default DB | `postgres` |

---

# ▶️ 2. Start the Database

From the directory containing `docker-compose.yml`:

```bash
docker compose up -d
```

Check logs:

```bash
docker logs -f ngelmak-pgdb
```

Stop:

```bash
docker compose down
```

---

# 🛠 3. Connect to PostgreSQL

### From inside the container

```bash
docker exec -it ngelmak-pgdb bash
psql -U admin postgres
```

### From your host machine

```bash
psql -h localhost -U admin -d postgres
```

---

# 🗄️ 4. Create Databases for Services

Each microservice should have its own database.

### Example: Auth service

```sql
CREATE DATABASE ngelmakauthdb OWNER admin;
```

### Example: Core service

```sql
CREATE DATABASE ngelmakcoredb OWNER admin;
```

List databases:

```sql
\l
```

---

# 👤 5. Create Application Users

Each service should have its own DB user.

### Create a user

```sql
CREATE USER auth_user WITH PASSWORD 'authpass';
CREATE USER core_user WITH PASSWORD 'corepass';
```

List users:

```sql
\du
```

---

# 🔑 6. Grant Privileges to Users

Switch to the service database:

```sql
\c ngelmakauthdb
```

Grant privileges:

```sql
GRANT CONNECT ON DATABASE ngelmakauthdb TO auth_user;
GRANT USAGE ON SCHEMA public TO auth_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO auth_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO auth_user;
```

Repeat for the core service:

```sql
\c ngelmakcoredb
GRANT CONNECT ON DATABASE ngelmakcoredb TO core_user;
GRANT USAGE ON SCHEMA public TO core_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO core_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO core_user;
```

---

# 🧹 7. Optional: Reset or Remove Users

Drop a user:

```sql
DROP USER auth_user;
```

Drop a database:

```sql
DROP DATABASE ngelmakauthdb;
```

---

# 📦 8. Data Persistence

All data is stored in:

```
./pgdata/
```

This folder contains:

- databases  
- roles  
- WAL logs  
- configuration  

Deleting this folder resets the entire database instance.

---

# 🧪 9. Quick Test Queries

Check connection:

```sql
SELECT version();
```

Create a test table:

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO test (name) VALUES ('hello');
SELECT * FROM test;
```