# Troubleshooting Guide - Ablage System

## 🎯 Überblick

Dieser Guide hilft bei der Diagnose und Behebung häufiger Probleme im Ablage-System. Strukturiert nach Symptomen mit klaren Schritt-für-Schritt-Lösungen.

---

## 📊 Quick Reference - Häufigste Probleme

| Problem | Wahrscheinliche Ursache | Quick Fix | Seite |
|---------|------------------------|-----------|-------|
| OCR processing failed | GPU OOM, Model nicht geladen | Restart Backend, Check VRAM | [Link](#ocr-processing-failures) |
| GPU out of memory | Batch zu groß, Memory Leak | Reduce batch size, Restart | [Link](#gpu-out-of-memory) |
| Database connection error | Connection pool exhausted | Increase pool size | [Link](#database-connection-errors) |
| High queue length | Backend offline, Overload | Scale backends | [Link](#high-queue-length) |
| Authentication failures | Token expired, Wrong credentials | Refresh token, Check DB | [Link](#authentication-failures) |
| Slow processing | Wrong backend, Resource starved | Check routing, Scale up | [Link](#slow-processing-times) |
| File upload fails | Size limit, Format unsupported | Check limits, Validate format | [Link](#file-upload-failures) |

---

## 🔴 CRITICAL ISSUES

### OCR Processing Failures

#### Symptom
```
ERROR: Processing failed for document ID: 12345
Status: FAILED
Error: RuntimeError: CUDA out of memory
```

#### Diagnose

**1. Check Backend Health**
```bash
# API Health Check
curl http://localhost:8000/api/v1/health

# Expected Response:
{
  "status": "healthy",
  "backends": {
    "deepseek": true,
    "got_ocr": true,
    "cpu_surya": true
  },
  "active_gpu_backend": "deepseek"
}
```

**2. Check GPU Memory**
```bash
# On GPU server
nvidia-smi

# Look for:
# - Memory Usage (should be < 90%)
# - Running processes
# - GPU temperature
```

**3. Check Logs**
```bash
# Backend logs
tail -f /var/log/ablage-system/backend.log

# Filter for errors
grep "ERROR" /var/log/ablage-system/backend.log | tail -50
```

#### Solution

**Option A: Quick Fix (Restart Backend)**
```bash
# Via systemd
sudo systemctl restart ablage-gpu-backend

# Via Docker
docker restart ablage-gpu-backend

# Via Kubernetes
kubectl rollout restart deployment/gpu-backend-deepseek -n ablage-system
```

**Option B: Clear GPU Memory**
```bash
# Kill Python processes using GPU
sudo pkill -9 python

# Clear CUDA cache
sudo rm -rf /tmp/cuda-*

# Restart NVIDIA driver (last resort)
sudo systemctl restart nvidia-persistenced
```

**Option C: Switch to CPU Backend**
```bash
# Temporary routing to CPU
curl -X POST http://localhost:8000/api/v1/admin/switch-gpu-backend \
  -H "Content-Type: application/json" \
  -d '{"backend": "cpu_surya"}'
```

**Option D: Reduce Batch Size**
```python
# In backend/config.py
MAX_BATCH_SIZE = {
    'deepseek': 2,  # Reduced from 4
    'got_ocr': 4,   # Reduced from 8
    'cpu_surya': 3  # Reduced from 5
}
```

---

### GPU Out of Memory

#### Symptom
```
torch.cuda.OutOfMemoryError: CUDA out of memory.
Tried to allocate 2.00 GiB (GPU 0; 15.70 GiB total capacity)
```

#### Diagnose

**1. Check Current VRAM Usage**
```bash
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv

# Expected: < 14 GB used (on 16GB GPU)
# If > 15 GB: PROBLEM
```

**2. Check for Memory Leaks**
```bash
# Monitor VRAM over time
watch -n 1 nvidia-smi

# If memory increases continuously without processing: MEMORY LEAK
```

**3. Check Active Processes**
```bash
# List processes using GPU
nvidia-smi pmon

# Expected: 1-2 Python processes
# If > 5 processes: TOO MANY WORKERS
```

#### Solution

**Option A: Emergency Cleanup**
```bash
# 1. Stop all GPU processes
sudo systemctl stop ablage-gpu-backend

# 2. Kill zombie processes
sudo fuser -k /dev/nvidia*

# 3. Clear CUDA cache
sudo nvidia-smi --gpu-reset

# 4. Restart backend
sudo systemctl start ablage-gpu-backend
```

**Option B: Optimize Model Loading**
```python
# backend/gpu/deepseek_backend.py

# Use quantization
self.model = AutoModelForVision2Seq.from_pretrained(
    self.model_name,
    torch_dtype=torch.float16,  # Instead of float32
    device_map="auto",
    load_in_8bit=True,  # 8-bit quantization
    trust_remote_code=True
)
```

**Option C: Implement Automatic Cleanup**
```python
# backend/core/gpu_manager.py

import torch
import gc

def cleanup_gpu_memory():
    """Force garbage collection and clear CUDA cache"""
    gc.collect()
    torch.cuda.empty_cache()
    torch.cuda.synchronize()

# Call after each batch
cleanup_gpu_memory()
```

**Option D: Enable Memory Monitoring**
```python
# Add to backend startup

import torch

# Set memory allocation config
torch.cuda.set_per_process_memory_fraction(0.9, 0)  # Use max 90% VRAM
os.environ['PYTORCH_CUDA_ALLOC_CONF'] = 'max_split_size_mb:512'
```

---

### Database Connection Errors

#### Symptom
```
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError)
FATAL: remaining connection slots are reserved
```

#### Diagnose

**1. Check Active Connections**
```sql
-- Connect to PostgreSQL
psql -U ablage_admin -d ablage_system

-- Check connection count
SELECT count(*) FROM pg_stat_activity;

-- Check by application
SELECT application_name, count(*)
FROM pg_stat_activity
GROUP BY application_name;

-- Check connection limit
SHOW max_connections;
```

**2. Check Connection Pool**
```python
# In Python shell
from backend.core.database import engine

print(f"Pool size: {engine.pool.size()}")
print(f"Checked out: {engine.pool.checkedout()}")
print(f"Overflow: {engine.pool.overflow()}")
```

**3. Check for Hanging Connections**
```sql
-- Find long-running queries
SELECT pid, usename, application_name, state, query_start, query
FROM pg_stat_activity
WHERE state != 'idle'
AND query_start < NOW() - INTERVAL '5 minutes'
ORDER BY query_start;
```

#### Solution

**Option A: Increase Connection Pool**
```python
# backend/core/database.py

from sqlalchemy import create_engine

engine = create_engine(
    DATABASE_URL,
    pool_size=20,           # Increased from 10
    max_overflow=40,        # Increased from 20
    pool_timeout=30,
    pool_recycle=3600,      # Recycle connections after 1h
    pool_pre_ping=True      # Verify connections
)
```

**Option B: Kill Idle Connections**
```sql
-- Kill connections idle for > 10 minutes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'ablage_system'
AND state = 'idle'
AND state_change < NOW() - INTERVAL '10 minutes';
```

**Option C: Increase PostgreSQL max_connections**
```bash
# Edit postgresql.conf
sudo nano /etc/postgresql/16/main/postgresql.conf

# Change:
max_connections = 200  # Increased from 100

# Restart PostgreSQL
sudo systemctl restart postgresql
```

**Option D: Implement Connection Pooling (PgBouncer)**
```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
ablage_system = host=localhost port=5432 dbname=ablage_system

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

---

### High Queue Length

#### Symptom
```
WARNING: Queue depth critical
Queue: ablage_processing
Depth: 1547 jobs
Oldest job: 45 minutes
```

#### Diagnose

**1. Check Queue Status**
```python
# Via Redis CLI
redis-cli

# Check queue length
LLEN ablage:processing:queue

# Check oldest job
LRANGE ablage:processing:queue 0 0
```

**2. Check Backend Availability**
```bash
# Check all backends
curl http://localhost:8000/api/v1/health | jq '.backends'

# Expected: all true
# If any false: Backend down
```

**3. Check Processing Rate**
```python
# Via Prometheus/Grafana
# Metric: processing_rate_per_minute

# Expected: > 60 documents/min
# If < 20 documents/min: Too slow
```

#### Solution

**Option A: Scale Up Backends**
```bash
# Kubernetes
kubectl scale deployment/cpu-backend-surya --replicas=10 -n ablage-system

# Docker Compose
docker-compose up -d --scale cpu-backend=5
```

**Option B: Restart Stuck Workers**
```bash
# Check for stuck workers
celery -A backend.tasks inspect active

# Restart workers
celery -A backend.tasks control shutdown
celery -A backend.tasks worker --loglevel=info &
```

**Option C: Prioritize Queue**
```python
# Process high-priority jobs first
from backend.tasks import reorder_queue

reorder_queue(priority='high')
```

**Option D: Emergency Queue Drain**
```python
# Move to CPU backend temporarily
from backend.orchestrator.router import router

router.override_backend = 'cpu_surya'  # Fast processing
```

---

### Authentication Failures

#### Symptom
```
HTTP 401 Unauthorized
{"detail": "Could not validate credentials"}
```

#### Diagnose

**1. Check Token Validity**
```bash
# Decode JWT token (without verification)
echo "YOUR_TOKEN" | jwt decode -

# Check expiry
# exp: 1234567890 (Unix timestamp)
date -d @1234567890
```

**2. Check Redis Token Store**
```bash
redis-cli

# Check if token is blacklisted
GET blacklist:token:YOUR_TOKEN_ID

# Check session
GET session:USER_ID
```

**3. Check User Status**
```sql
-- Connect to database
psql -U ablage_admin -d ablage_system

-- Check user
SELECT id, email, is_active, is_verified, last_login
FROM users
WHERE email = 'user@example.com';
```

#### Solution

**Option A: Refresh Token**
```bash
# Get new token
curl -X POST http://localhost:8000/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "YOUR_REFRESH_TOKEN"}'
```

**Option B: Clear Token Blacklist**
```bash
# If token incorrectly blacklisted
redis-cli DEL blacklist:token:TOKEN_ID
```

**Option C: Reactivate User**
```sql
-- If user deactivated
UPDATE users
SET is_active = true, is_verified = true
WHERE email = 'user@example.com';
```

**Option D: Reset Password**
```bash
# Via API
curl -X POST http://localhost:8000/api/v1/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'
```

---

## 🟡 PERFORMANCE ISSUES

### Slow Processing Times

#### Symptom
```
Average processing time: 45 seconds per page
Expected: < 5 seconds per page
```

#### Diagnose

**1. Check Backend Selection**
```bash
# Check which backend is being used
tail -f /var/log/ablage-system/backend.log | grep "Backend selected"

# If "cpu_surya" for simple docs: WRONG BACKEND
```

**2. Check Resource Utilization**
```bash
# GPU
nvidia-smi --query-gpu=utilization.gpu,utilization.memory --format=csv

# CPU
top -b -n 1 | head -20

# If GPU util < 50%: Not using GPU efficiently
# If CPU util > 90%: CPU bottleneck
```

**3. Profile Processing**
```python
# Add profiling
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

# Your processing code here

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)
```

#### Solution

**Option A: Fix Backend Routing**
```python
# backend/orchestrator/router.py

def _select_backend(self, complexity, doc_type, force_backend=None):
    # Ensure simple docs go to fast backend
    if complexity == DocumentComplexity.SIMPLE:
        return BackendType.GOT_OCR  # Fast OCR
    elif complexity == DocumentComplexity.COMPLEX:
        return BackendType.DEEPSEEK  # Accurate but slower
    else:
        return BackendType.CPU_SURYA
```

**Option B: Enable Batch Processing**
```python
# Process multiple pages in one batch
from backend.services.ocr import batch_process_document

result = batch_process_document(
    document_id,
    batch_size=8  # Process 8 pages at once
)
```

**Option C: Optimize Preprocessing**
```python
# Disable unnecessary preprocessing
PREPROCESSING_CONFIG = {
    'deskew': False,          # Disable if not needed
    'noise_reduction': False,  # Disable for clean scans
    'enhancement': False       # Disable for good quality
}
```

**Option D: Use Caching**
```python
# Enable result caching
CACHE_CONFIG = {
    'enabled': True,
    'ttl': 86400,  # 24 hours
    'backend': 'redis'
}
```

---

### High Memory Usage (RAM)

#### Symptom
```
System memory: 58GB / 64GB (90%)
OOM Killer activated
```

#### Diagnose

**1. Check Memory by Process**
```bash
# Top memory consumers
ps aux --sort=-%mem | head -20

# Detailed memory breakdown
smem -tk -P python
```

**2. Check for Memory Leaks**
```python
# Install memory_profiler
pip install memory_profiler

# Profile function
from memory_profiler import profile

@profile
def process_document(doc):
    # Your code
    pass
```

**3. Monitor Over Time**
```bash
# Watch memory usage
watch -n 5 'free -h && ps aux --sort=-%mem | head -10'
```

#### Solution

**Option A: Limit Worker Memory**
```python
# backend/config.py

WORKER_CONFIG = {
    'max_memory_per_worker': 8 * 1024 * 1024 * 1024,  # 8GB
    'restart_on_memory_limit': True
}
```

**Option B: Implement Garbage Collection**
```python
import gc

# Force garbage collection after heavy operations
def process_batch(images):
    results = []
    for img in images:
        result = process_image(img)
        results.append(result)

        # Clear references
        del img

    # Force cleanup
    gc.collect()
    return results
```

**Option C: Use Memory-Efficient Loading**
```python
# Load images lazily
from PIL import Image

def lazy_load_images(paths):
    for path in paths:
        with Image.open(path) as img:
            # Process immediately
            yield process_image(img)
            # Image closed automatically
```

**Option D: Increase System Swap**
```bash
# Create 16GB swap file
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 🟢 OPERATIONAL ISSUES

### File Upload Failures

#### Symptom
```
HTTP 413 Payload Too Large
or
HTTP 415 Unsupported Media Type
```

#### Diagnose

**1. Check File Size**
```bash
# Get file size
ls -lh document.pdf

# Check upload limit
curl -I http://localhost:8000/api/v1/ocr/process | grep -i "limit"
```

**2. Check File Type**
```bash
# Get MIME type
file --mime-type document.pdf

# Expected: application/pdf, image/png, image/jpeg
```

**3. Check Nginx Limits**
```bash
# Check nginx config
sudo nginx -T | grep client_max_body_size
```

#### Solution

**Option A: Increase Upload Limit**
```python
# backend/api/main.py

from fastapi import FastAPI

app = FastAPI()

# Increase limit to 100MB
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["*"],
    max_upload_size=100 * 1024 * 1024
)
```

**Option B: Update Nginx Config**
```nginx
# /etc/nginx/sites-available/ablage-system

server {
    client_max_body_size 100M;  # Increase from default
    client_body_timeout 300s;    # Increase timeout

    location /api/v1/ocr/process {
        proxy_pass http://backend;
        proxy_request_buffering off;  # Stream uploads
    }
}
```

**Option C: Add File Validation**
```python
# backend/api/v1/documents.py

ALLOWED_EXTENSIONS = {'pdf', 'png', 'jpg', 'jpeg', 'tiff'}
MAX_FILE_SIZE = 100 * 1024 * 1024  # 100MB

def validate_file(file: UploadFile):
    # Check extension
    ext = file.filename.split('.')[-1].lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(415, f"File type {ext} not supported")

    # Check size (if available)
    if file.size and file.size > MAX_FILE_SIZE:
        raise HTTPException(413, "File too large")
```

**Option D: Implement Chunked Upload**
```python
# For very large files
@app.post("/api/v1/ocr/upload-chunk")
async def upload_chunk(
    chunk: UploadFile,
    chunk_number: int,
    total_chunks: int,
    upload_id: str
):
    # Save chunk
    chunk_path = f"/tmp/uploads/{upload_id}/chunk_{chunk_number}"
    with open(chunk_path, "wb") as f:
        f.write(await chunk.read())

    # If last chunk, merge all
    if chunk_number == total_chunks - 1:
        merge_chunks(upload_id, total_chunks)
```

---

### Redis Connection Issues

#### Symptom
```
redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379.
Connection refused.
```

#### Diagnose

**1. Check Redis Status**
```bash
# Check if running
sudo systemctl status redis

# Check port
sudo netstat -tlnp | grep 6379
```

**2. Test Connection**
```bash
# Try to connect
redis-cli ping

# Expected: PONG
# If error: Redis not responding
```

**3. Check Logs**
```bash
# Redis logs
sudo tail -f /var/log/redis/redis-server.log

# Look for errors
```

#### Solution

**Option A: Restart Redis**
```bash
sudo systemctl restart redis
sudo systemctl status redis
```

**Option B: Check Configuration**
```bash
# Edit redis.conf
sudo nano /etc/redis/redis.conf

# Check:
bind 127.0.0.1  # Or 0.0.0.0 for remote access
port 6379
protected-mode yes

# Restart after changes
sudo systemctl restart redis
```

**Option C: Clear Redis Memory**
```bash
redis-cli

# Check memory usage
INFO memory

# If > 90% full:
FLUSHDB  # Clear current DB (WARNING: deletes data!)

# Or increase maxmemory
CONFIG SET maxmemory 4gb
```

**Option D: Connection Pool Settings**
```python
# backend/core/cache.py

import redis

redis_pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=50,  # Increase pool
    socket_connect_timeout=5,
    socket_timeout=5,
    retry_on_timeout=True
)

redis_client = redis.Redis(connection_pool=redis_pool)
```

---

## 🛠️ DEBUGGING TECHNIQUES

### Enabling Debug Logging

```python
# backend/core/logging.py

import logging
import sys

# Set to DEBUG
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/ablage-system/debug.log'),
        logging.StreamHandler(sys.stdout)
    ]
)

# Enable SQL query logging
logging.getLogger('sqlalchemy.engine').setLevel(logging.DEBUG)

# Enable HTTP request logging
logging.getLogger('uvicorn.access').setLevel(logging.DEBUG)
```

### Request Tracing

```python
# Add request ID to all logs

import uuid
from fastapi import Request

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id

    # Add to logs
    logging.info(f"[{request_id}] {request.method} {request.url}")

    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

### Performance Profiling

```bash
# Install py-spy
pip install py-spy

# Profile running process
sudo py-spy top --pid $(pgrep -f "uvicorn backend.main")

# Generate flame graph
sudo py-spy record -o profile.svg --pid $(pgrep -f "uvicorn backend.main")
```

---

## 📋 System Recovery Procedures

### Full System Restart (Minimal Downtime)

```bash
#!/bin/bash
# recovery.sh

echo "Starting system recovery..."

# 1. Stop processing new jobs
curl -X POST http://localhost:8000/api/v1/admin/maintenance-mode \
  -d '{"enabled": true}'

# 2. Wait for current jobs to finish (max 10 min)
timeout 600 bash -c 'until [ $(curl -s http://localhost:8000/api/v1/queue/depth) -eq 0 ]; do sleep 5; done'

# 3. Backup current state
pg_dump -U ablage_admin ablage_system > backup_$(date +%Y%m%d_%H%M%S).sql
redis-cli BGSAVE

# 4. Restart services
sudo systemctl restart postgresql
sudo systemctl restart redis
sudo systemctl restart ablage-gpu-backend
sudo systemctl restart ablage-cpu-backend
sudo systemctl restart ablage-api

# 5. Health check
sleep 30
curl http://localhost:8000/api/v1/health

# 6. Disable maintenance mode
curl -X POST http://localhost:8000/api/v1/admin/maintenance-mode \
  -d '{"enabled": false}'

echo "Recovery complete!"
```

### Database Recovery

```bash
# 1. Stop application
sudo systemctl stop ablage-api ablage-gpu-backend ablage-cpu-backend

# 2. Restore from backup
psql -U ablage_admin -d ablage_system < backup_20240115_143000.sql

# 3. Run migrations (if needed)
cd /opt/ablage-system
source venv/bin/activate
alembic upgrade head

# 4. Verify data
psql -U ablage_admin -d ablage_system -c "SELECT COUNT(*) FROM documents;"

# 5. Restart application
sudo systemctl start ablage-api ablage-gpu-backend ablage-cpu-backend
```

---

## 📞 Escalation Procedures

### Level 1: Self-Service (This Guide)
- ✅ Common issues with documented solutions
- ✅ System restarts
- ✅ Configuration changes
- ⏱️ Response Time: Immediate

### Level 2: Engineering Team
**Contact**: engineering@ablage-system.com
**When**: Issue not resolved after 30 minutes
**Required Info**:
- Error messages (full stack trace)
- System logs (last 100 lines)
- Steps to reproduce
- System metrics at time of error

### Level 3: Platform Team
**Contact**: platform-team@ablage-system.com
**When**: Infrastructure-level issues (AWS, Kubernetes)
**Required Info**:
- All Level 2 info
- Terraform state
- Kubernetes events
- Cloud provider metrics

### Level 4: On-Call Engineer
**Contact**: +49 XXX XXXXXXX (PagerDuty)
**When**: System down > 15 minutes, data loss risk
**Severity**: P0 - Critical

---

## 📚 Additional Resources

- [System Architecture](./Mission.md)
- [Deployment Guide](./Deployment-Production.md)
- [Monitoring & Metrics](./Grafana_Config.md)
- [Database Guide](./Database.md)
- [API Documentation](./API/API_Documentation.md)

---

## 🔄 Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-01-15 | Initial troubleshooting guide |
