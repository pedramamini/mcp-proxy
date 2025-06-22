# **MCP DevTools: Deployment Guide**

## **Quick Start**

### **Prerequisites**

- Docker and Docker Compose
- Python 3.10+ (for local development)
- Node.js 18+ (for frontend development)
- PostgreSQL 15+ or SQLite (for database)
- Redis 7+ (for real-time messaging)

### **Production Deployment (Docker Compose)**

1. **Clone and Configure**
```bash
git clone <repository-url>
cd mcp-devtools
cp .env.example .env
# Edit .env with your configuration
```

2. **Start Services**
```bash
docker-compose up -d
```

3. **Initialize Database**
```bash
docker-compose exec mcp-devtools python -m alembic upgrade head
```

4. **Access Dashboard**
- Web UI: http://localhost:3000
- API Docs: http://localhost:3000/docs
- Health Check: http://localhost:3000/health

## **Configuration Files**

### **Environment Variables (.env)**

```bash
# Application Settings
APP_NAME=MCP DevTools
APP_VERSION=1.0.0
DEBUG=false
LOG_LEVEL=INFO
ENABLE_INSPECTOR=true
WEB_UI_ENABLED=true

# Database Configuration
DATABASE_URL=postgresql://mcpuser:mcppass@postgres:5432/mcpdevtools
# Alternative for SQLite: DATABASE_URL=sqlite+aiosqlite:///./mcpdevtools.db

# Redis Configuration
REDIS_URL=redis://redis:6379/0

# Security Settings
JWT_SECRET_KEY=your-super-secret-jwt-key-change-this-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=1440

# CORS Settings
CORS_ORIGINS=http://localhost:3000,https://your-domain.com
CORS_ALLOW_CREDENTIALS=true

# Performance Settings
MAX_CONCURRENT_CONNECTIONS=100
MESSAGE_RETENTION_DAYS=30
WEBSOCKET_PING_INTERVAL=30
MAX_MESSAGE_SIZE=1048576

# MCP Server Configuration
MCP_SERVER_CONFIG_PATH=/app/config/servers.json
DEFAULT_SERVER_TIMEOUT=60

# Monitoring
ENABLE_METRICS=true
METRICS_PORT=9090
HEALTH_CHECK_INTERVAL=30
```

### **Docker Compose Configuration**

```yaml
# docker-compose.yml
version: '3.8'

services:
  mcp-devtools:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: production
    ports:
      - "8080:8080"  # MCP proxy port
      - "3000:3000"  # Web UI port
      - "9090:9090"  # Metrics port
    environment:
      - DATABASE_URL=postgresql://mcpuser:mcppass@postgres:5432/mcpdevtools
      - REDIS_URL=redis://redis:6379/0
      - LOG_LEVEL=INFO
      - ENABLE_INSPECTOR=true
      - WEB_UI_ENABLED=true
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./config:/app/config:ro
      - ./logs:/app/logs
      - mcp_data:/app/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mcpdevtools
      POSTGRES_USER: mcpuser
      POSTGRES_PASSWORD: mcppass
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mcpuser -d mcpdevtools"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./docker/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/ssl:/etc/nginx/ssl:ro
      - ./logs/nginx:/var/log/nginx
    depends_on:
      - mcp-devtools
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  mcp_data:
```

### **Development Docker Compose**

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  mcp-devtools-dev:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: development
    ports:
      - "8080:8080"
      - "3000:3000"
      - "9090:9090"
    environment:
      - DATABASE_URL=postgresql://mcpuser:mcppass@postgres:5432/mcpdevtools_dev
      - REDIS_URL=redis://redis:6379/1
      - LOG_LEVEL=DEBUG
      - DEBUG=true
      - ENABLE_INSPECTOR=true
    volumes:
      - .:/app
      - /app/node_modules
      - dev_data:/app/data
    depends_on:
      - postgres
      - redis
    command: ["python", "-m", "uvicorn", "src.mcp_proxy.web.app:app", "--reload", "--host", "0.0.0.0", "--port", "3000"]

  postgres:
    extends:
      file: docker-compose.yml
      service: postgres
    environment:
      POSTGRES_DB: mcpdevtools_dev

  redis:
    extends:
      file: docker-compose.yml
      service: redis

volumes:
  dev_data:
```

## **Dockerfile Configuration**

```dockerfile
# docker/Dockerfile
FROM python:3.11-slim as base

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Create app user
RUN useradd --create-home --shell /bin/bash app

# Set work directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Development stage
FROM base as development

# Install development dependencies
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt

# Install Node.js for frontend development
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs

USER app
EXPOSE 3000 8080 9090

# Production stage
FROM base as production

# Copy application code
COPY --chown=app:app . .

# Install production dependencies only
RUN pip install --no-deps .

# Build frontend assets
RUN cd src/mcp_proxy/web/static && \
    npm ci --only=production && \
    npm run build

# Create necessary directories
RUN mkdir -p /app/logs /app/data /app/config && \
    chown -R app:app /app

USER app

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

EXPOSE 3000 8080 9090

CMD ["python", "-m", "uvicorn", "src.mcp_proxy.web.app:app", "--host", "0.0.0.0", "--port", "3000"]
```

## **Nginx Configuration**

```nginx
# docker/nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream mcp_devtools {
        server mcp-devtools:3000;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=websocket:10m rate=5r/s;

    server {
        listen 80;
        server_name localhost;

        # Redirect HTTP to HTTPS in production
        # return 301 https://$server_name$request_uri;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Referrer-Policy strict-origin-when-cross-origin;

        # Static files
        location /static/ {
            alias /app/static/;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # WebSocket proxy
        location /ws {
            limit_req zone=websocket burst=10 nodelay;

            proxy_pass http://mcp_devtools;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket timeout settings
            proxy_read_timeout 86400;
            proxy_send_timeout 86400;
        }

        # API endpoints
        location /api/ {
            limit_req zone=api burst=20 nodelay;

            proxy_pass http://mcp_devtools;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Main application
        location / {
            proxy_pass http://mcp_devtools;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Health check
        location /health {
            proxy_pass http://mcp_devtools/health;
            access_log off;
        }
    }

    # HTTPS configuration (uncomment for production)
    # server {
    #     listen 443 ssl http2;
    #     server_name your-domain.com;
    #
    #     ssl_certificate /etc/nginx/ssl/cert.pem;
    #     ssl_certificate_key /etc/nginx/ssl/key.pem;
    #     ssl_protocols TLSv1.2 TLSv1.3;
    #     ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    #     ssl_prefer_server_ciphers off;
    #
    #     # Include the same location blocks as above
    # }
}
```

## **Database Configuration**

### **PostgreSQL Initialization**

```sql
-- docker/postgres/init.sql
-- Create additional indexes for performance
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_messages_timestamp_desc ON messages (timestamp DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_messages_server_status ON messages (server_name, status);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_messages_method_timestamp ON messages (method, timestamp DESC);

-- Create partitioning for large datasets (optional)
-- This is useful if you expect millions of messages
-- CREATE TABLE messages_y2024m01 PARTITION OF messages
-- FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Create stored procedures for analytics
CREATE OR REPLACE FUNCTION get_message_stats(
    start_date TIMESTAMP DEFAULT NOW() - INTERVAL '24 hours',
    end_date TIMESTAMP DEFAULT NOW()
)
RETURNS TABLE (
    total_messages BIGINT,
    unique_servers BIGINT,
    avg_processing_time NUMERIC,
    error_rate NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        COUNT(*) as total_messages,
        COUNT(DISTINCT server_name) as unique_servers,
        AVG(processing_time_ms) as avg_processing_time,
        (COUNT(*) FILTER (WHERE status = 'error')::NUMERIC / COUNT(*) * 100) as error_rate
    FROM messages
    WHERE timestamp BETWEEN start_date AND end_date;
END;
$$ LANGUAGE plpgsql;
```

### **Redis Configuration**

```conf
# docker/redis/redis.conf
# Basic configuration
bind 0.0.0.0
port 6379
timeout 300
keepalive 60

# Memory management
maxmemory 256mb
maxmemory-policy allkeys-lru

# Persistence (adjust based on your needs)
save 900 1
save 300 10
save 60 10000

# Security
requirepass your-redis-password-change-this

# Logging
loglevel notice
logfile /var/log/redis/redis-server.log

# Performance tuning
tcp-backlog 511
databases 16
```

## **Monitoring and Observability**

### **Prometheus Configuration**

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'mcp-devtools'
    static_configs:
      - targets: ['mcp-devtools:9090']
    scrape_interval: 5s
    metrics_path: /metrics

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

### **Grafana Dashboard Configuration**

```json
{
  "dashboard": {
    "title": "MCP DevTools Dashboard",
    "panels": [
      {
        "title": "Message Throughput",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mcp_messages_total[5m])",
            "legendFormat": "Messages/sec"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(mcp_message_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "Active Connections",
        "type": "singlestat",
        "targets": [
          {
            "expr": "mcp_websocket_connections_active",
            "legendFormat": "WebSocket Connections"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mcp_errors_total[5m])",
            "legendFormat": "Errors/sec"
          }
        ]
      }
    ]
  }
}
```

## **Security Configuration**

### **SSL/TLS Setup**

```bash
# Generate self-signed certificates for development
mkdir -p docker/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout docker/nginx/ssl/key.pem \
    -out docker/nginx/ssl/cert.pem \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"

# For production, use Let's Encrypt or your certificate authority
```

### **Authentication Setup**

```python
# Create initial admin user
python scripts/create_admin_user.py --username admin --email admin@example.com
```

### **Firewall Configuration**

```bash
# UFW configuration for Ubuntu
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# For development, also allow:
sudo ufw allow 3000/tcp  # Web UI
sudo ufw allow 8080/tcp  # MCP Proxy
```

## **Backup and Recovery**

### **Database Backup Script**

```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="mcpdevtools"

# Create backup directory
mkdir -p $BACKUP_DIR

# PostgreSQL backup
docker-compose exec -T postgres pg_dump -U mcpuser $DB_NAME | gzip > $BACKUP_DIR/postgres_$TIMESTAMP.sql.gz

# Redis backup
docker-compose exec -T redis redis-cli --rdb - | gzip > $BACKUP_DIR/redis_$TIMESTAMP.rdb.gz

# Configuration backup
tar -czf $BACKUP_DIR/config_$TIMESTAMP.tar.gz config/

# Clean old backups (keep last 7 days)
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $TIMESTAMP"
```

### **Recovery Script**

```bash
#!/bin/bash
# scripts/restore.sh

if [ -z "$1" ]; then
    echo "Usage: $0 <backup_timestamp>"
    exit 1
fi

TIMESTAMP=$1
BACKUP_DIR="/backups"

# Stop services
docker-compose down

# Restore PostgreSQL
gunzip -c $BACKUP_DIR/postgres_$TIMESTAMP.sql.gz | docker-compose exec -T postgres psql -U mcpuser mcpdevtools

# Restore Redis
gunzip -c $BACKUP_DIR/redis_$TIMESTAMP.rdb.gz | docker-compose exec -T redis redis-cli --pipe

# Restore configuration
tar -xzf $BACKUP_DIR/config_$TIMESTAMP.tar.gz

# Start services
docker-compose up -d

echo "Restore completed: $TIMESTAMP"
```

## **Performance Tuning**

### **Database Optimization**

```sql
-- PostgreSQL performance tuning
-- Add to postgresql.conf

# Memory settings
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB

# Connection settings
max_connections = 100
shared_preload_libraries = 'pg_stat_statements'

# Logging
log_statement = 'all'
log_duration = on
log_min_duration_statement = 1000

# Checkpoints
checkpoint_completion_target = 0.9
wal_buffers = 16MB
```

### **Application Performance**

```python
# Performance configuration in .env
UVICORN_WORKERS=4
UVICORN_WORKER_CLASS=uvicorn.workers.UvicornWorker
MAX_CONCURRENT_CONNECTIONS=1000
CONNECTION_POOL_SIZE=20
CONNECTION_POOL_MAX_OVERFLOW=30
WEBSOCKET_PING_INTERVAL=30
MESSAGE_BATCH_SIZE=100
CACHE_TTL=300
```

## **Troubleshooting**

### **Common Issues and Solutions**

1. **WebSocket Connection Failures**
```bash
# Check nginx configuration
docker-compose exec nginx nginx -t

# Check application logs
docker-compose logs mcp-devtools

# Test WebSocket connection
wscat -c ws://localhost/ws
```

2. **Database Connection Issues**
```bash
# Check PostgreSQL status
docker-compose exec postgres pg_isready -U mcpuser

# Check connection from application
docker-compose exec mcp-devtools python -c "
from sqlalchemy import create_engine
engine = create_engine('postgresql://mcpuser:mcppass@postgres:5432/mcpdevtools')
print('Connection successful' if engine.connect() else 'Connection failed')
"
```

3. **High Memory Usage**
```bash
# Monitor memory usage
docker stats

# Check for memory leaks
docker-compose exec mcp-devtools python -c "
import psutil
process = psutil.Process()
print(f'Memory usage: {process.memory_info().rss / 1024 / 1024:.2f} MB')
"
```

4. **Performance Issues**
```bash
# Check database performance
docker-compose exec postgres psql -U mcpuser -d mcpdevtools -c "
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
"

# Monitor Redis performance
docker-compose exec redis redis-cli info stats
```

### **Log Analysis**

```bash
# View application logs
docker-compose logs -f mcp-devtools

# Search for errors
docker-compose logs mcp-devtools | grep ERROR

# Monitor WebSocket connections
docker-compose logs mcp-devtools | grep "WebSocket"

# Check database queries
docker-compose logs postgres | grep "LOG:"
```

This deployment guide provides comprehensive instructions for setting up MCP DevTools in both development and production environments, with proper monitoring, security, and backup procedures.