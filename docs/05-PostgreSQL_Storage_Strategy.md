# CrawlerX PostgreSQL 统一存储策略
## Unified PostgreSQL Storage Strategy

> **版本**: v2.0.0
> **日期**: 2025-12-25
> **项目代号**: CrawlerX
> **文档状态**: 正式发布

---

## 📋 文档信息

| 项目 | 内容 |
|------|------|
| **文档名称** | CrawlerX PostgreSQL 统一存储策略 |
| **文档版本** | v2.0.0 |
| **存储方案** | PostgreSQL 15 (全功能统一存储) |
| **创建日期** | 2025-12-25 |

---

## 1. 存储策略概述

### 1.1 统一存储架构

```
┌─────────────────────────────────────────────────────────────────┐
│                   CrawlerX 统一 PostgreSQL 架构                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Application Layer                     │   │
│  │  (Vision Service | Data Service | Task Service | ...)   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    PostgreSQL 15                         │   │
│  │                                                             │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │   │
│  │  │ 关系数据表   │ │ JSONB文档表  │ │ 文件存储表   │     │   │
│  │  │ (Tables)     │ │ (JSONB)      │ │ (BYTEA/LO)   │     │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘     │   │
│  │                                                             │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │   │
│  │  │ 会话缓存表   │ │ 任务队列表   │ │ 日志分区表   │     │   │
│  │  │ (UNLOGGED)   │ │ (SKIP LOCKED)│ │ (Partitioned)│     │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘     │   │
│  │                                                             │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │   │
│  │  │ 分布式锁     │ │ 计数器       │ │ 时序数据     │     │   │
│  │  │ (Advisory)   │ │ (Sequences)  │ │ (Timescale)  │     │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘     │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    PostgreSQL Extensions                 │   │
│  │  • pg_trgm (模糊搜索)  • btree_gin (复合索引)            │   │
│  │  • pg_partman (自动分区)  • pg_stat_statements (监控)    │   │
│  │  • TimescaleDB (时序数据扩展)                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 存储方案对比

| 原方案 | PostgreSQL 统一方案 | 迁移策略 | 性能影响 |
|--------|---------------------|----------|----------|
| **Redis** | UNLOGGED 表 + 查询优化 | 会话/缓存 → 表存储 | 略慢但可接受 |
| **MinIO** | BYTEA + 文件系统路径 | 大文件 → 文件系统，元数据 → PG | 无影响 |
| **MongoDB** | JSONB + 分区表 | 日志 → JSONB 表 | 相近或更好 |

### 1.3 统一存储优势

| 优势 | 说明 |
|------|------|
| **简化运维** | 单一数据库，减少运维复杂度 |
| **降低成本** | 无需维护多个存储系统 |
| **ACID 事务** | 跨表事务保证数据一致性 |
| **统一查询** | SQL 联表查询，无需跨系统 |
| **备份恢复** | 统一备份策略，简化恢复流程 |
| **减少依赖** | 减少外部服务依赖 |

---

## 2. PostgreSQL 功能映射

### 2.1 Redis 功能迁移

#### 2.1.1 会话存储

**原 Redis 方案**:
```redis
HSET session:abc123 user_id "uuid" email "user@example.com"
EXPIRE session:abc123 86400
```

**PostgreSQL 方案**:
```sql
-- 会话表 (使用 UNLOGGED 提升性能，可接受会话丢失)
CREATE UNLOGGED TABLE sessions (
    id VARCHAR(64) PRIMARY KEY,
    user_id UUID NOT NULL,
    session_data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- 索引
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
CREATE INDEX idx_sessions_user_id ON sessions(user_id);

-- TTL 清理 (pg_cron 或应用层)
DELETE FROM sessions WHERE expires_at < CURRENT_TIMESTAMP;
```

#### 2.1.2 缓存存储

**PostgreSQL 方案**:
```sql
-- 缓存表 (UNLOGGED，性能优先)
CREATE UNLOGGED TABLE cache (
    key VARCHAR(255) PRIMARY KEY,
    value JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP WITH TIME ZONE
);

-- 索引
CREATE INDEX idx_cache_expires_at ON cache(expires_at);

-- 查询示例
SELECT value FROM cache
WHERE key = 'user:profile:123'
  AND (expires_at IS NULL OR expires_at > CURRENT_TIMESTAMP);
```

#### 2.1.3 任务队列

**PostgreSQL 方案**:
```sql
-- 任务队列表
CREATE TABLE task_queue (
    id SERIAL PRIMARY KEY,
    queue_name VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL,
    priority INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending',
                    -- CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    attempts INTEGER DEFAULT 0,
    max_attempts INTEGER DEFAULT 3,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    reserved_at TIMESTAMP WITH TIME ZONE,
    processed_at TIMESTAMP WITH TIME ZONE
);

-- 索引
CREATE INDEX idx_queue_status ON task_queue(status, priority, created_at);
CREATE INDEX idx_queue_reserved ON task_queue(reserved_at) WHERE status = 'processing';

-- 出队 (FOR UPDATE SKIP LOCKED)
UPDATE task_queue
SET status = 'processing',
    reserved_at = CURRENT_TIMESTAMP,
    attempts = attempts + 1
WHERE id = (
    SELECT id FROM task_queue
    WHERE status = 'pending'
      AND (attempts < max_attempts OR max_attempts = 0)
    ORDER BY priority DESC, created_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
RETURNING *;

-- 入队
INSERT INTO task_queue (queue_name, payload, priority)
VALUES ('extraction', '{"task_id": "xxx"}', 1);

-- 完成任务
UPDATE task_queue
SET status = 'completed',
    processed_at = CURRENT_TIMESTAMP
WHERE id = :task_id;
```

#### 2.1.4 分布式锁

**PostgreSQL 方案**:
```sql
-- 方式1: 使用 Advisory Lock (推荐)
SELECT pg_advisory_lock(hashint8('task:lock:123'));
-- 执行业务逻辑
SELECT pg_advisory_unlock(hashint8('task:lock:123'));

-- 方式2: 使用表锁
CREATE TABLE distributed_locks (
    lock_key VARCHAR(255) PRIMARY KEY,
    locked_by VARCHAR(100),
    locked_at TIMESTAMP WITH TIME ZONE,
    expires_at TIMESTAMP WITH TIME ZONE
);

-- 获取锁
INSERT INTO distributed_locks (lock_key, locked_by, locked_at, expires_at)
VALUES ('task:lock:123', 'worker-1', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP + INTERVAL '5 minutes')
ON CONFLICT (lock_key) DO NOTHING
RETURNING lock_key;

-- 释放锁
DELETE FROM distributed_locks
WHERE lock_key = 'task:lock:123'
  AND locked_by = 'worker-1';
```

#### 2.1.5 计数器与限流

**PostgreSQL 方案**:
```sql
-- 计数器表
CREATE UNLOGGED TABLE counters (
    key VARCHAR(255) PRIMARY KEY,
    value BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 原子递增
INSERT INTO counters (key, value)
VALUES ('user:123:api_calls', 1)
ON CONFLICT (key) DO UPDATE
SET value = counters.value + 1,
    updated_at = CURRENT_TIMESTAMP
RETURNING value;

-- 限流 (窗口期计数)
CREATE UNLOGGED TABLE rate_limits (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL,
    window_start TIMESTAMP WITH TIME ZONE NOT NULL,
    request_count INTEGER DEFAULT 1
);

-- 使用索引去重实现限流
CREATE UNIQUE INDEX idx_rate_limits_unique
ON rate_limits (user_id, window_start);

-- 检查并递增
WITH window AS (
    SELECT date_trunc('minute', CURRENT_TIMESTAMP) AS window_start
)
INSERT INTO rate_limits (user_id, window_start)
SELECT 'user-uuid', window_start FROM window
ON CONFLICT (user_id, window_start) DO UPDATE
SET request_count = rate_limits.request_count + 1
RETURNING request_count <= 60 AS allowed; -- 每分钟60次
```

### 2.2 MinIO 功能迁移

#### 2.2.1 文件存储策略

**混合方案**:
- **小文件** (<1MB): 存储 PostgreSQL BYTEA 字段
- **大文件** (≥1MB): 存储文件系统，PostgreSQL 存储路径

```sql
-- 文件存储表
CREATE TABLE stored_files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50) NOT NULL,
                    -- 'screenshot', 'snapshot', 'export', etc.
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(100),

    -- 存储方式
    storage_type VARCHAR(20) NOT NULL,
                       -- CHECK (storage_type IN ('database', 'filesystem')),
    file_path VARCHAR(500),  -- 文件系统路径 (当 storage_type = 'filesystem')
    file_data BYTEA,         -- 文件二进制 (当 storage_type = 'database')

    -- 元数据
    metadata JSONB DEFAULT '{}',

    -- 关联信息
    related_object_type VARCHAR(50),
    related_object_id UUID,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 索引
CREATE INDEX idx_files_type ON stored_files(file_type);
CREATE INDEX idx_files_related ON stored_files(related_object_type, related_object_id);
CREATE INDEX idx_files_created ON stored_files(created_at);

-- 示例: 插入小文件
INSERT INTO stored_files (file_name, file_type, file_size, mime_type, storage_type, file_data)
VALUES ('screenshot.png', 'screenshot', 102400, 'image/png', 'database', '\x...');

-- 示例: 插入大文件
INSERT INTO stored_files (file_name, file_type, file_size, mime_type, storage_type, file_path)
VALUES ('snapshot.png', 'snapshot', 5242880, 'image/png', 'filesystem', '/data/crawlerx/screenshots/2025/01/xxx.png');
```

#### 2.2.2 文件系统目录结构

```
/data/crawlerx/
├── screenshots/
│   └── {year}/
│       └── {month}/
│           └── {day}/
│               └── {uuid}.png
├── snapshots/
│   └── {year}/
│       └── {month}/
│           └── {day}/
│               └── {uuid}/
│                   ├── snapshot.html
│                   └── screenshot.png
└── exports/
    └── {year}/
        └── {month}/
            └── {task_id}_{format}.{ext}
```

### 2.3 MongoDB 功能迁移

#### 2.3.1 日志存储 (JSONB)

**PostgreSQL 方案**:
```sql
-- 操作日志表 (替代 MongoDB operation_logs)
CREATE UNLOGGED TABLE operation_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    log_level VARCHAR(20) NOT NULL,
                     -- CHECK (log_level IN ('DEBUG', 'INFO', 'WARN', 'ERROR')),

    -- 用户信息
    user_id UUID,
    organization_id UUID,

    -- 操作信息
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id UUID,

    -- 请求/响应 (JSONB)
    request JSONB,
    response JSONB,

    -- 性能指标
    duration_ms INTEGER,
    db_query_count INTEGER,

    -- 上下文
    context JSONB DEFAULT '{}',

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 分区 (按月)
CREATE TABLE operation_logs_partitioned (
    LIKE operation_logs INCLUDING ALL
) PARTITION BY RANGE (created_at);

-- 创建分区
CREATE TABLE operation_logs_2025_01 PARTITION OF operation_logs_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- 索引 (JSONB 索引)
CREATE INDEX idx_op_logs_user ON operation_logs(user_id, created_at DESC);
CREATE INDEX idx_op_logs_action ON operation_logs(action);
CREATE INDEX idx_op_logs_resource ON operation_logs(resource_type, resource_id);
CREATE INDEX idx_op_logs_request ON operation_logs USING GIN (request);
CREATE INDEX idx_op_logs_context ON operation_logs USING GIN (context);

-- TTL (使用 pg_partman 或 pg_cron 自动清理)
DELETE FROM operation_logs WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '90 days';
```

#### 2.3.2 错误日志

```sql
-- 错误日志表
CREATE TABLE error_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 关联信息
    task_id UUID,
    target_id UUID,
    user_id UUID,

    -- 错误信息
    error_type VARCHAR(50) NOT NULL,
    error_code VARCHAR(50),
    error_message TEXT NOT NULL,
    stack_trace TEXT,

    -- 上下文 (JSONB)
    context JSONB DEFAULT '{}',

    -- 处理状态
    retry_count INTEGER DEFAULT 0,
    is_resolved BOOLEAN DEFAULT false,
    resolved_at TIMESTAMP WITH TIME ZONE,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 分区
CREATE TABLE error_logs_partitioned (
    LIKE error_logs INCLUDING ALL
) PARTITION BY RANGE (created_at);

-- 索引
CREATE INDEX idx_error_logs_task ON error_logs(task_id);
CREATE INDEX idx_error_logs_type ON error_logs(error_type);
CREATE INDEX idx_error_logs_resolved ON error_logs(is_resolved);
CREATE INDEX idx_error_logs_context ON error_logs USING GIN (context);

-- TTL
DELETE FROM error_logs WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '30 days';
```

#### 2.3.3 性能指标 (时序数据)

**PostgreSQL 方案**:
```sql
-- 性能指标表 (可选 TimescaleDB 扩展)
CREATE TABLE performance_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 服务信息
    service_name VARCHAR(50) NOT NULL,
    endpoint VARCHAR(100),

    -- 指标数据 (JSONB)
    metrics JSONB NOT NULL DEFAULT '{}',
              -- {
              --   "request_count": 1000,
              --   "success_count": 950,
              --   "error_count": 50,
              --   "avg_duration_ms": 150,
              --   "p50_duration_ms": 120,
              --   "p95_duration_ms": 300,
              --   "p99_duration_ms": 500
              -- }

    -- 资源使用 (JSONB)
    resource_usage JSONB DEFAULT '{}',

    timestamp TIMESTAMP WITH TIME ZONE NOT NULL
);

-- 分区 (按天)
CREATE TABLE performance_metrics_partitioned (
    LIKE performance_metrics INCLUDING ALL
) PARTITION BY RANGE (timestamp);

-- 创建分区
CREATE TABLE performance_metrics_2025_01_25 PARTITION OF performance_metrics_partitioned
    FOR VALUES FROM ('2025-01-25') TO ('2025-01-26');

-- 索引
CREATE INDEX idx_perf_metrics_service ON performance_metrics(service_name, timestamp DESC);
CREATE INDEX idx_perf_metrics_timestamp ON performance_metrics(timestamp DESC);
CREATE INDEX idx_perf_metrics_data ON performance_metrics USING GIN (metrics);

-- 聚合查询示例
SELECT
    date_trunc('hour', timestamp) AS hour,
    service_name,
    AVG((metrics->>'avg_duration_ms')::INTEGER) AS avg_duration,
    SUM((metrics->>'request_count')::INTEGER) AS total_requests
FROM performance_metrics
WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY 1, 2
ORDER BY 1, 2;
```

---

## 3. PostgreSQL 扩展配置

### 3.1 必需扩展

```sql
-- 创建扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";        -- UUID 生成
CREATE EXTENSION IF NOT EXISTS "pg_trgm";         -- 模糊搜索
CREATE EXTENSION IF NOT EXISTS "btree_gin";       -- 复合索引
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements"; -- 查询性能分析
CREATE EXTENSION IF NOT EXISTS "pg_cron";         -- 定时任务 (可选)
```

### 3.2 可选扩展 (按需启用)

```sql
-- TimescaleDB (时序数据扩展)
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- pg_partman (自动分区管理)
CREATE EXTENSION IF NOT EXISTS pg_partman;

-- postgis (地理信息，如果需要)
CREATE EXTENSION IF NOT EXISTS postgis;
```

### 3.3 配置优化

```sql
-- postgresql.conf 关键配置

# 内存配置
shared_buffers = 4GB              # 总内存的 25%
effective_cache_size = 12GB       # 总内存的 75%
work_mem = 64MB                   # 单个操作内存
maintenance_work_mem = 1GB

# 连接配置
max_connections = 200
max_worker_processes = 16
max_parallel_workers_per_gather = 4
max_parallel_workers = 16

# WAL 配置
wal_buffers = 16MB
min_wal_size = 2GB
max_wal_size = 8GB
wal_compression = on

# 查询优化
random_page_cost = 1.1            # SSD 优化
effective_io_concurrency = 200    # SSD 优化

# 日志配置
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_min_duration_statement = 1000  # 记录慢查询 (>1s)
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

---

## 4. 性能优化策略

### 4.1 索引优化

```sql
-- B-tree 索引 (默认)
CREATE INDEX idx_users_email ON users(email);

-- GIN 索引 (JSONB)
CREATE INDEX idx_sessions_data ON sessions USING GIN (session_data);

-- GiST 索引 (范围、全文搜索)
CREATE INDEX idx_sessions_expires_gist ON sessions USING GIST (expires_at);

-- 表达式索引
CREATE INDEX idx_tasks_status_created ON extraction_tasks(status, created_at DESC);

-- 部分索引 (WHERE 条件)
CREATE INDEX idx_active_tasks ON extraction_tasks(status)
WHERE status IN ('pending', 'running');
```

### 4.2 查询优化

```sql
-- 使用 EXPLAIN ANALYZE 分析查询
EXPLAIN ANALYZE
SELECT * FROM extraction_tasks
WHERE user_id = 'xxx'
  AND status = 'pending'
ORDER BY created_at DESC
LIMIT 10;

-- 使用 CTE (Common Table Expressions) 优化复杂查询
WITH user_tasks AS (
    SELECT id, name, created_at
    FROM extraction_tasks
    WHERE user_id = 'xxx'
),
task_stats AS (
    SELECT task_id, COUNT(*) AS target_count
    FROM task_targets
    WHERE task_id IN (SELECT id FROM user_tasks)
    GROUP BY task_id
)
SELECT t.*, ts.target_count
FROM user_tasks t
LEFT JOIN task_stats ts ON ts.task_id = t.id;
```

### 4.3 连接池配置

```python
# 使用连接池 (如 SQLAlchemy + psycopg2)
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:pass@localhost/crawlerx',
    poolclass=QueuePool,
    pool_size=20,           # 连接池大小
    max_overflow=40,        # 最大溢出连接
    pool_pre_ping=True,     # 连接健康检查
    pool_recycle=3600       # 连接回收时间 (秒)
)
```

---

## 5. 备份与恢复

### 5.1 备份策略

```bash
#!/bin/bash
# PostgreSQL 全量备份

pg_dump -h localhost -U crawlerx -d crawlerx \
  --format=custom \
  --compress=9 \
  --file=/backup/crawlerx_$(date +%Y%m%d).dump

# 仅备份 Schema
pg_dump -h localhost -U crawlerx -d crawlerx \
  --schema-only \
  --file=/backup/crawlerx_schema.sql

# 仅备份特定表
pg_dump -h localhost -U crawlerx -d crawlerx \
  --table=extraction_tasks \
  --table=extraction_results \
  --file=/backup/crawlerx_tables.sql
```

### 5.2 恢复策略

```bash
#!/bin/bash
# 恢复数据库

pg_restore -h localhost -U crawlerx -d crawlerx_new \
  --clean --if-exists \
  --jobs=4 \
  /backup/crawlerx_20251225.dump
```

### 5.3 PITR (Point-In-Time Recovery)

```bash
# 配置 WAL 归档
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /wal_archive/%f'

# 恢复到指定时间点
pg_ctl start -D /var/lib/postgresql/data -o "-o 'restore_command=\"cp /wal_archive/%f %p\"'"
```

---

## 6. 监控与维护

### 6.1 监控指标

```sql
-- 连接数监控
SELECT count(*) FROM pg_stat_activity;

-- 慢查询监控
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- 表膨胀监控
SELECT schemaname, tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
  pg_stat_get_dead_tuples(c.oid) AS dead_tuples
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE relkind = 'r'
ORDER BY dead_tuples DESC;

-- 索引使用监控
SELECT schemaname, tablename, indexname,
  idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

### 6.2 定期维护

```sql
-- 定期 VACUUM ANALYZE
VACUUM ANALYZE extraction_tasks;
VACUUM ANALYZE extraction_results;

-- 清理过期数据
DELETE FROM operation_logs WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '90 days';
DELETE FROM sessions WHERE expires_at < CURRENT_TIMESTAMP;
DELETE FROM cache WHERE expires_at < CURRENT_TIMESTAMP;

-- 重建索引 (可选)
REINDEX TABLE CONCURRENTLY extraction_tasks;
REINDEX TABLE CONCURRENTLY extraction_results;
```

---

## 7. 数据迁移计划

### 7.1 从 Redis 迁移

```sql
-- 迁移会话数据
CREATE TABLE sessions_backup AS
SELECT * FROM redis_sessions;

-- 迁移缓存数据
INSERT INTO cache (key, value, expires_at)
SELECT key, value::JSONB, NOW() + INTERVAL 'ttl seconds'
FROM redis_cache;
```

### 7.2 从 MinIO 迁移

```sql
-- 仅迁移元数据到 PostgreSQL
INSERT INTO stored_files (file_name, file_type, file_size, mime_type, storage_type, file_path)
SELECT
    file_name,
    file_type,
    file_size,
    mime_type,
    'filesystem',
    '/data/crawlerx/' || file_path
FROM minio_files_metadata;
```

### 7.3 从 MongoDB 迁移

```sql
-- 迁移操作日志
INSERT INTO operation_logs (
    log_level, user_id, action, resource_type, resource_id,
    request, response, duration_ms, context, created_at
)
SELECT
    log_level,
    user_id::UUID,
    action,
    resource_type,
    resource_id::UUID,
    request::JSONB,
    response::JSONB,
    duration_ms,
    context::JSONB,
    created_at
FROM mongodb_operation_logs;
```

---

## 8. 附录

### 8.1 性能对比

| 操作 | Redis | PostgreSQL (UNLOGGED) | 差异 |
|------|-------|----------------------|------|
| 读写 (QPS) | 100,000+ | 10,000-50,000 | 2-10倍 |
| 延迟 (ms) | <1 | 1-5 | 可接受 |
| 内存占用 | 高 | 低 | 更优 |

### 8.2 存储容量规划

| 数据类型 | 日增量 | 年增量 | 3年容量 |
|----------|--------|--------|---------|
| 业务数据 | 1GB | 365GB | 1TB |
| 日志数据 | 5GB | 1.8TB | 5.5TB |
| 文件数据 | 10GB | 3.6TB | 11TB |
| **总计** | **16GB** | **5.8TB** | **17.5TB** |

### 8.3 推荐配置

| 规模 | CPU | 内存 | 存储 | 适用场景 |
|------|-----|------|------|----------|
| 小型 | 4核 | 16GB | 500GB SSD | 开发/测试 |
| 中型 | 8核 | 32GB | 2TB SSD | 小型生产 |
| 大型 | 16核+ | 64GB+ | 10TB+ SSD | 大规模生产 |

---

**文档变更记录**

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|----------|------|
| v2.0.0 | 2025-12-25 | 统一 PostgreSQL 存储策略 | AI System Architect |

---

*本文档定义了 CrawlerX 项目统一使用 PostgreSQL 进行数据存储的完整策略，包括会话、缓存、队列、文件、日志等各类数据的存储方案。*
