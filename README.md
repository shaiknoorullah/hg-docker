# HalalGoes Infrastructure Stack

This is a standalone Docker Compose setup for the HalalGoes platform infrastructure. It includes all necessary services: PostgreSQL, Redis Sentinel, MinIO, Temporal, and the HalalGoes API.

## Prerequisites

- Docker Engine 20.10+
- Docker Compose v2.0+
- At least 4GB RAM available for Docker
- Ports available: 3456, 5050, 5432, 6379, 6432, 7233, 8080, 9000, 9001, 9080, 26379, 26380

## Quick Start

### 1. Configure Environment Variables

Copy the example environment file and configure it:

```bash
cp .env.example .env
```

Edit `.env` and set the **required** variables:

```bash
# REQUIRED - Must be set
HG_DB_PASS=your_secure_password_here
PGADMIN_PASS=your_pgadmin_password_here
MINIO_ROOT_PASSWORD=your_minio_password_here
HOST_IP=192.168.1.100  # Replace with your machine's IP address
```

**Finding Your Host IP:**
- Linux: `ip addr show | grep "inet " | grep -v 127.0.0.1`
- macOS: `ifconfig | grep "inet " | grep -v 127.0.0.1`
- Windows: `ipconfig` (look for IPv4 Address)

### 2. Start the Stack

```bash
docker-compose up -d
```

### 3. Verify Services

Check that all services are running:

```bash
docker-compose ps
```

All services should show status as "Up" or "Up (healthy)".

## Services

| Service | Port | Description | Access URL |
|---------|------|-------------|------------|
| **HalalGoes API** | 3456 | Main REST API | http://localhost:3456 |
| **WebSocket** | 9080 | Real-time updates | ws://localhost:9080 |
| **PostgreSQL** | 5432 | Primary database | localhost:5432 |
| **PGBouncer** | 6432 | Connection pooler | localhost:6432 |
| **PGAdmin** | 5050 | Database GUI | http://localhost:5050 |
| **Redis Master** | 6379 | Cache & sessions | localhost:6379 |
| **Redis Sentinels** | 26379, 26380 | HA monitoring | localhost:26379 |
| **RedisInsight** | 5540 | Redis GUI | http://localhost:5540 |
| **MinIO API** | 9000 | Object storage | http://localhost:9000 |
| **MinIO Console** | 9001 | Storage GUI | http://localhost:9001 |
| **Temporal** | 7233 | Workflow engine | localhost:7233 |
| **Temporal UI** | 8080 | Workflow dashboard | http://localhost:8080 |

## API Version Management

### Using Specific API Version

By default, the stack uses the latest API version. To use a specific version:

```bash
# In .env file
API_VERSION=1.2.25
```

Or via command line:

```bash
API_VERSION=1.2.25 docker-compose up -d api
```

### Available Versions

All published versions are available on Docker Hub:
https://hub.docker.com/r/devsupreme0/halalgoes-api/tags

### Updating the API

```bash
# Pull latest version
docker-compose pull api

# Restart API service
docker-compose up -d api
```

## Database Access

### Using PGAdmin

1. Open http://localhost:5050
2. Login with credentials from `.env`:
   - Email: `PGADMIN_EMAIL` (default: admin@example.com)
   - Password: `PGADMIN_PASS`
3. Add server:
   - Host: `postgres-halalgoes`
   - Port: `5432`
   - Database: `HG_DB` (default: halalgoes_db)
   - Username: `HG_DB_USER` (default: postgres)
   - Password: `HG_DB_PASS`

### Using psql

```bash
docker exec -it postgres-halalgoes psql -U postgres -d halalgoes_db
```

## Storage Management

### MinIO Console

1. Open http://localhost:9001
2. Login with credentials from `.env`:
   - Username: `MINIO_ROOT_USER` (default: admin)
   - Password: `MINIO_ROOT_PASSWORD`

### Creating Buckets

MinIO buckets need to be created manually for file uploads:

```bash
# Access MinIO container
docker exec -it hg-file-storage mc alias set local http://localhost:9000 admin YOUR_MINIO_PASSWORD

# Create required buckets
docker exec -it hg-file-storage mc mb local/profile-pictures
docker exec -it hg-file-storage mc mb local/restaurant-images
docker exec -it hg-file-storage mc mb local/food-images
```

## Redis Monitoring

### RedisInsight

1. Open http://localhost:5540
2. Add database:
   - Host: `redis-master`
   - Port: `6379`
   - Name: `HalalGoes Redis`

### Check Sentinel Status

```bash
docker exec -it sentinel-1 redis-cli -p 26379 SENTINEL master mymaster
```

## Temporal Workflows

### Temporal UI

Access the Temporal dashboard at http://localhost:8080 to:
- Monitor workflow executions
- View workflow history
- Debug failed workflows
- Search workflow runs

## Logs

### View logs for all services
```bash
docker-compose logs -f
```

### View logs for specific service
```bash
docker-compose logs -f api
docker-compose logs -f postgres
docker-compose logs -f redis-master
```

## Troubleshooting

### API Service Not Starting

Check logs:
```bash
docker-compose logs api
```

Common issues:
- Database migrations failed: Check PostgreSQL is healthy
- Temporal connection failed: Ensure Temporal service is running
- Redis connection failed: Verify Redis master is up

### Database Connection Issues

1. Check PostgreSQL health:
```bash
docker-compose ps postgres
```

2. Test direct connection:
```bash
docker exec -it postgres-halalgoes pg_isready -U postgres
```

3. Check PGBouncer:
```bash
docker-compose logs pgbouncer
```

### Redis Failover Not Working

1. Check Sentinel status:
```bash
docker exec -it sentinel-1 redis-cli -p 26379 SENTINEL master mymaster
```

2. Verify HOST_IP is correct:
```bash
echo $HOST_IP
# Should match your machine's IP, not 127.0.0.1
```

### Temporal Namespace Issues

The API uses the `default` namespace. If you see namespace errors:

```bash
# Check Temporal is healthy
docker-compose logs temporal

# Verify namespace exists
docker exec -it temporal-admin-tools tctl --namespace default namespace describe
```

## Stopping Services

### Stop all services
```bash
docker-compose down
```

### Stop and remove volumes (⚠️ deletes all data)
```bash
docker-compose down -v
```

## Backup & Restore

### Backup PostgreSQL Database

```bash
docker exec -it postgres-halalgoes pg_dump -U postgres halalgoes_db > backup.sql
```

### Restore PostgreSQL Database

```bash
cat backup.sql | docker exec -i postgres-halalgoes psql -U postgres -d halalgoes_db
```

### Backup MinIO Data

```bash
docker run --rm -v minio_data:/data -v $(pwd):/backup alpine tar czf /backup/minio_backup.tar.gz /data
```

## Network Architecture

All services communicate on the `halalgoes-network` bridge network with static IPs:

- `172.21.0.3` - Redis Master
- `172.21.0.4` - Redis Slave 1
- `172.21.0.5` - Redis Slave 2
- `172.21.0.6` - Sentinel 1
- `172.21.0.7` - Sentinel 2
- `172.21.0.9` - RedisInsight
- `172.21.0.10` - PostgreSQL
- `172.21.0.11` - PGBouncer
- `172.21.0.12` - PGAdmin
- `172.21.0.13` - Temporal
- `172.21.0.14` - Temporal Admin Tools
- `172.21.0.15` - Temporal UI
- `172.21.0.20` - MinIO
- `172.21.0.100` - HalalGoes API

## Support

For issues with:
- **Infrastructure setup**: Check this README and troubleshooting section
- **API functionality**: Contact the backend team
- **API versions**: Check Docker Hub for available versions

## License

UNLICENSED - Private use only
