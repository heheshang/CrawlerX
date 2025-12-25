# CrawlerX 数据库设计文档 (SQLite + PostgreSQL)
## Database Design Document for Tauri Desktop App

> **版本**: v1.0.0
> **日期**: 2025-12-25
> **项目代号**: CrawlerX
> **架构类型**: Tauri 2.0 桌面应用 (CS 架构)

---

## 📋 文档信息

| 项目 | 内容 |
|------|------|
| **文档名称** | CrawlerX 数据库设计文档 (SQLite + PostgreSQL) |
| **文档版本** | v1.0.0 |
| **创建日期** | 2025-12-25 |
| **架构类型** | 本地 SQLite + 可选云端 PostgreSQL 同步 |

---

## 1. 数据库概述

### 1.1 存储架构

```
┌─────────────────────────────────────────────────────────────────┐
│                   CrawlerX 数据存储架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           本地存储 (SQLite) - 主存储                       │   │
│  │                                                         │   │
│  │  文件位置: ~/Library/Application Support/CrawlerX/        │   │
│  │            %APPDATA%/CrawlerX/                             │   │
│  │            ~/.config/CrawlerX/                             │   │
│  │                                                         │   │
│  │  • 用户配置      • 任务数据      • 提取结果                │   │
│  │  • 标记模板      • 本地缓存      • 会话数据                │   │
│  │  • 导出文件      • 日志记录      • 系统设置                │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                    (可选云端同步)                               │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │        云端存储 (PostgreSQL) - 备份/同步                   │   │
│  │                                                         │   │
│  │  • 跨设备数据同步  • 云端备份      • 数据恢复              │   │
│  │  • 团队协作        • 多设备访问    • 历史归档              │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 数据库选型

| 数据库 | 用途 | 版本 | 理由 |
|--------|------|------|------|
| **SQLite** | 本地主存储 | 3.x+ | 嵌入式、零配置、跨平台 |
| **PostgreSQL** | 云端同步 (可选) | 15.x | 高可用、团队协作 |

### 1.3 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 表名 | snake_case, 复数 | `extraction_tasks`, `task_targets` |
| 字段名 | snake_case | `task_id`, `created_at` |
| 索引名 | `idx_表名_字段名` | `idx_tasks_status` |
| 主键名 | `id` (TEXT UUID) | - |
| 时间戳 | Unix timestamp (INTEGER) | `created_at` |

---

## 2. SQLite 本地数据库设计

### 2.1 核心表结构

#### 2.1.1 extraction_tasks (提取任务表)

```sql
CREATE TABLE extraction_tasks (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,

    -- 任务类型
    task_type TEXT NOT NULL CHECK(task_type IN ('single', 'batch', 'scheduled')),

    -- 配置 (JSON)
    config TEXT NOT NULL,

    -- 调度配置
    schedule_cron TEXT,
    schedule_timezone TEXT DEFAULT 'UTC',
    next_run_at INTEGER,

    -- 状态
    status TEXT NOT NULL CHECK(status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),

    -- 统计
    total_targets INTEGER DEFAULT 0,
    completed_targets INTEGER DEFAULT 0,
    failed_targets INTEGER DEFAULT 0,

    -- 时间戳 (Unix timestamp)
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    started_at INTEGER,
    completed_at INTEGER
);

-- 索引
CREATE INDEX idx_tasks_status ON extraction_tasks(status);
CREATE INDEX idx_tasks_next_run ON extraction_tasks(next_run_at) WHERE next_run_at IS NOT NULL;
CREATE INDEX idx_tasks_created_at ON extraction_tasks(created_at DESC);

-- 触发器: 自动更新 updated_at
CREATE TRIGGER update_tasks_timestamp
AFTER UPDATE ON extraction_tasks
FOR EACH ROW
BEGIN
    UPDATE extraction_tasks SET updated_at = strftime('%s', 'now') WHERE id = NEW.id;
END;
```

#### 2.1.2 task_targets (任务目标表)

```sql
CREATE TABLE task_targets (
    id TEXT PRIMARY KEY,
    task_id TEXT NOT NULL REFERENCES extraction_tasks(id) ON DELETE CASCADE,

    -- 目标信息
    url TEXT NOT NULL,
    url_hash TEXT NOT NULL,

    -- 截图信息
    screenshot_path TEXT,
    screenshot_width INTEGER,
    screenshot_height INTEGER,

    -- 标记信息 (JSON)
    markers TEXT NOT NULL,

    -- 状态
    status TEXT NOT NULL CHECK(status IN ('pending', 'processing', 'completed', 'failed', 'skipped')),

    -- 结果关联
    result_id TEXT REFERENCES extraction_results(id),

    -- 错误信息
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    last_retry_at INTEGER,

    -- 时间戳
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    started_at INTEGER,
    completed_at INTEGER
);

-- 索引
CREATE INDEX idx_targets_task_id ON task_targets(task_id);
CREATE INDEX idx_targets_url_hash ON task_targets(url_hash);
CREATE INDEX idx_targets_status ON task_targets(status);
CREATE INDEX idx_targets_result_id ON task_targets(result_id);

-- 触发器
CREATE TRIGGER update_targets_timestamp
AFTER UPDATE ON task_targets
FOR EACH ROW
BEGIN
    UPDATE task_targets SET updated_at = strftime('%s', 'now') WHERE id = NEW.id;
END;
```

#### 2.1.3 extraction_results (提取结果表)

```sql
CREATE TABLE extraction_results (
    id TEXT PRIMARY KEY,

    -- 关联
    target_id TEXT NOT NULL REFERENCES task_targets(id),
    task_id TEXT NOT NULL REFERENCES extraction_tasks(id),

    -- 源信息
    source_url TEXT NOT NULL,

    -- 提取方法
    extraction_method TEXT NOT NULL CHECK(extraction_method IN ('html', 'api', 'hybrid', 'manual')),

    -- 提取数据 (JSON)
    data TEXT NOT NULL,

    -- 快照
    html_snapshot_path TEXT,
    api_snapshots TEXT,

    -- 质量指标
    confidence REAL,
    validation_status TEXT CHECK(validation_status IN ('valid', 'invalid', 'unknown')),

    -- 性能
    extraction_duration_ms INTEGER,

    -- 版本
    version INTEGER DEFAULT 1,
    is_latest INTEGER DEFAULT 1,

    -- 时间戳
    extracted_at INTEGER NOT NULL,
    created_at INTEGER NOT NULL,

    UNIQUE(target_id, version)
);

-- 索引
CREATE INDEX idx_results_target_id ON extraction_results(target_id);
CREATE INDEX idx_results_task_id ON extraction_results(task_id);
CREATE INDEX idx_results_extracted_at ON extraction_results(extracted_at DESC);
CREATE INDEX idx_results_is_latest ON extraction_results(is_latest);
```

#### 2.1.4 marker_templates (标记模板表)

```sql
CREATE TABLE marker_templates (
    id TEXT PRIMARY KEY,

    -- 模板信息
    name TEXT NOT NULL,
    description TEXT,
    category TEXT,

    -- 适用范围
    url_pattern TEXT,
    domain_pattern TEXT,

    -- 标记配置 (JSON)
    markers TEXT NOT NULL,

    -- 统计
    usage_count INTEGER DEFAULT 0,
    success_rate REAL,

    -- 共享
    is_public INTEGER DEFAULT 0,
    is_system INTEGER DEFAULT 0,

    -- 状态
    status TEXT DEFAULT 'active' CHECK(status IN ('active', 'inactive', 'deprecated')),

    -- 时间戳
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- 索引
CREATE INDEX idx_templates_category ON marker_templates(category);
CREATE INDEX idx_templates_domain ON marker_templates(domain_pattern);
CREATE INDEX idx_templates_status ON marker_templates(status);

-- 触发器
CREATE TRIGGER update_templates_timestamp
AFTER UPDATE ON marker_templates
FOR EACH ROW
BEGIN
    UPDATE marker_templates SET updated_at = strftime('%s', 'now') WHERE id = NEW.id;
END;
```

#### 2.1.5 settings (系统设置表)

```sql
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    value_type TEXT DEFAULT 'string' CHECK(value_type IN ('string', 'integer', 'boolean', 'json')),

    -- 元数据
    description TEXT,
    category TEXT,

    -- 更新
    updated_at INTEGER NOT NULL
);

-- 索引
CREATE INDEX idx_settings_category ON settings(category);

-- 默认设置
INSERT INTO settings (key, value, value_type, category) VALUES
    ('max.concurrent_tasks', '3', 'integer', 'system'),
    ('browser.headless', 'true', 'boolean', 'browser'),
    ('browser.timeout', '30', 'integer', 'browser'),
    ('screenshot.quality', '90', 'integer', 'screenshot'),
    ('data.auto_export', 'false', 'boolean', 'export'),
    ('sync.enabled', 'false', 'boolean', 'sync');
```

#### 2.1.6 sync_queue (同步队列表 - 可选)

```sql
CREATE TABLE sync_queue (
    id TEXT PRIMARY KEY,
    table_name TEXT NOT NULL,
    operation TEXT NOT NULL CHECK(operation IN ('INSERT', 'UPDATE', 'DELETE')),
    record_id TEXT NOT NULL,
    record_data TEXT,

    -- 状态
    status TEXT DEFAULT 'pending' CHECK(status IN ('pending', 'synced', 'failed')),
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,

    -- 时间戳
    created_at INTEGER NOT NULL,
    synced_at INTEGER
);

-- 索引
CREATE INDEX idx_sync_queue_status ON sync_queue(status);
CREATE INDEX idx_sync_queue_table ON sync_queue(table_name, record_id);
```

### 2.2 辅助表

#### 2.2.1 local_cache (本地缓存表)

```sql
CREATE TABLE local_cache (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    value_type TEXT DEFAULT 'string',

    -- TTL
    expires_at INTEGER,

    -- 时间戳
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- 索引
CREATE INDEX idx_cache_expires ON local_cache(expires_at);

-- 自动清理
DELETE FROM local_cache WHERE expires_at < strftime('%s', 'now');
```

#### 2.2.2 audit_logs (审计日志表)

```sql
CREATE TABLE audit_logs (
    id TEXT PRIMARY KEY,

    -- 操作信息
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id TEXT,

    -- 变更 (JSON)
    changes TEXT,

    -- 结果
    status TEXT DEFAULT 'success' CHECK(status IN ('success', 'failure')),
    error_message TEXT,

    -- 时间戳
    created_at INTEGER NOT NULL
);

-- 索引
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);

-- 定期清理 (保留30天)
DELETE FROM audit_logs WHERE created_at < strftime('%s', 'now', '-30 days');
```

---

## 3. Rust 数据访问层

### 3.1 SQLx 配置

```toml
[dependencies]
# 数据库
sqlx = { version = "0.7", features = ["runtime-tokio", "sqlite", "uuid", "chrono", "json"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
```

### 3.2 数据模型

```rust
// src-tauri/src/repository/models.rs
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExtractionTask {
    pub id: String,
    pub name: String,
    pub description: Option<String>,
    pub task_type: String,
    pub config: TaskConfig,
    pub schedule_cron: Option<String>,
    pub schedule_timezone: Option<String>,
    pub next_run_at: Option<DateTime<Utc>>,
    pub status: String,
    pub total_targets: i32,
    pub completed_targets: i32,
    pub failed_targets: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TaskConfig {
    pub urls: Vec<String>,
    pub markers: Vec<Marker>,
    pub options: ExtractionOptions,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Marker {
    pub id: String,
    pub x: u32,
    pub y: u32,
    pub width: u32,
    pub height: u32,
    pub label: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExtractionOptions {
    pub wait_for_load: bool,
    pub capture_api: bool,
    pub timeout_seconds: u64,
}
```

### 3.3 Repository 实现

```rust
// src-tauri/src/repository/sqlite.rs
use sqlx::{SqlitePool, Row};
use anyhow::Result;
use crate::repository::models::ExtractionTask;

pub struct SqliteRepository {
    pool: SqlitePool,
}

impl SqliteRepository {
    pub async fn new(db_path: &str) -> Result<Self> {
        // 创建数据库目录
        if let Some(parent) = std::path::Path::new(db_path).parent() {
            std::fs::create_dir_all(parent)?;
        }

        let pool = SqlitePool::connect(db_path).await?;

        // 启用 WAL 模式
        sqlx::query("PRAGMA journal_mode=WAL")
            .execute(&pool)
            .await?;

        // 优化设置
        sqlx::query("PRAGMA synchronous=NORMAL")
            .execute(&pool)
            .await?;

        sqlx::query("PRAGMA cache_size=-64000")  -- 64MB
            .execute(&pool)
            .await?;

        // 运行迁移
        self::migrate::run_migrations(&pool).await?;

        Ok(Self { pool })
    }

    pub async fn create_task(
        &self,
        task: &NewTask,
    ) -> Result<String> {
        let id = Uuid::new_v4().to_string();
        let now = Utc::now();
        let config_json = serde_json::to_string(&task.config)?;

        sqlx::query(
            r#"
            INSERT INTO extraction_tasks
            (id, name, description, task_type, config, status, created_at, updated_at)
            VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8)
            "#
        )
        .bind(&id)
        .bind(&task.name)
        .bind(&task.description)
        .bind(&task.task_type)
        .bind(&config_json)
        .bind("pending")
        .bind(now.timestamp())
        .bind(now.timestamp())
        .execute(&self.pool)
        .await?;

        Ok(id)
    }

    pub async fn get_tasks(&self) -> Result<Vec<ExtractionTask>> {
        let tasks = sqlx::query_as::<_, ExtractionTask>(
            "SELECT * FROM extraction_tasks ORDER BY created_at DESC"
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(tasks)
    }

    pub async fn get_task_by_id(&self, id: &str) -> Result<Option<ExtractionTask>> {
        let task = sqlx::query_as::<_, ExtractionTask>(
            "SELECT * FROM extraction_tasks WHERE id = ?1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await?;

        Ok(task)
    }

    pub async fn update_task_status(
        &self,
        id: &str,
        status: &str,
    ) -> Result<()> {
        sqlx::query(
            "UPDATE extraction_tasks SET status = ?1, updated_at = ?2 WHERE id = ?3"
        )
        .bind(status)
        .bind(Utc::now().timestamp())
        .bind(id)
        .execute(&self.pool)
        .await?;

        Ok(())
    }
}
```

### 3.4 数据库迁移

```rust
// src-tauri/src/repository/migrate.rs
use sqlx::SqlitePool;
use anyhow::Result;

pub async fn run_migrations(pool: &SqlitePool) -> Result<()> {
    // 创建 extraction_tasks 表
    sqlx::query(
        r#"
        CREATE TABLE IF NOT EXISTS extraction_tasks (
            id TEXT PRIMARY KEY,
            name TEXT NOT NULL,
            description TEXT,
            task_type TEXT NOT NULL CHECK(task_type IN ('single', 'batch', 'scheduled')),
            config TEXT NOT NULL,
            schedule_cron TEXT,
            schedule_timezone TEXT DEFAULT 'UTC',
            next_run_at INTEGER,
            status TEXT NOT NULL CHECK(status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
            total_targets INTEGER DEFAULT 0,
            completed_targets INTEGER DEFAULT 0,
            failed_targets INTEGER DEFAULT 0,
            created_at INTEGER NOT NULL,
            updated_at INTEGER NOT NULL,
            started_at INTEGER,
            completed_at INTEGER
        );
        "#
    )
    .execute(pool)
    .await?;

    // 创建其他表...

    Ok(())
}
```

---

## 4. PostgreSQL 云端同步 (可选)

### 4.1 同步架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    数据同步架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐         ┌──────────────┐                    │
│  │  SQLite      │────────▶│ Sync Service │                    │
│  │  (本地)       │  上传   │  (检测变化)  │                    │
│  └──────────────┘         └──────────────┘                    │
│                                  │                               │
│                                  ▼                               │
│  ┌──────────────┐         ┌──────────────┐                    │
│  │  PostgreSQL  │◀────────│ Cloud Sync   │                    │
│  │  (云端)       │  下载   │   API        │                    │
│  └──────────────┘         └──────────────┘                    │
│                                                                 │
│  同步策略:                                                         │
│  • 实时同步: 检测本地变化立即上传                                 │
│  • 定期同步: 每5分钟检查一次                                      │
│  • 手动同步: 用户手动触发                                         │
│  • 冲突解决: 以最新更新为准                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 同步服务实现

```rust
// src-tauri/src/services/sync_service.rs
use sqlx::SqlitePool;
use reqwest::Client;

pub struct SyncService {
    local_pool: SqlitePool,
    http_client: Client,
    api_url: String,
}

impl SyncService {
    pub fn new(local_pool: SqlitePool, api_url: String) -> Self {
        Self {
            local_pool,
            http_client: Client::new(),
            api_url,
        }
    }

    pub async fn sync_tasks(&self) -> Result<(), SyncError> {
        // 1. 获取本地修改
        let local_changes = self.get_local_changes("extraction_tasks").await?;

        // 2. 上传到云端
        for change in local_changes {
            self.upload_change(&change).await?;
        }

        // 3. 从云端下载更新
        let remote_updates = self.fetch_remote_updates("extraction_tasks").await?;

        // 4. 应用到本地
        for update in remote_updates {
            self.apply_remote_update(&update).await?;
        }

        Ok(())
    }

    async fn get_local_changes(&self, table: &str) -> Result<Vec<SyncChange>> {
        let changes = sqlx::query_as::<_, SyncChange>(
            "SELECT * FROM sync_queue
             WHERE table_name = ?1 AND status = 'pending'
             ORDER BY created_at ASC"
        )
        .bind(table)
        .fetch_all(&self.local_pool)
        .await?;

        Ok(changes)
    }
}

#[derive(Debug, Clone)]
pub struct SyncChange {
    pub id: String,
    pub table_name: String,
    pub operation: String,
    pub record_id: String,
    pub record_data: String,
}
```

---

## 5. 数据备份与恢复

### 5.1 本地备份

```rust
// src-tauri/src/services/backup_service.rs
use std::path::PathBuf;

pub struct BackupService {
    db_path: PathBuf,
    backup_dir: PathBuf,
}

impl BackupService {
    pub fn new(db_path: PathBuf, backup_dir: PathBuf) -> Self {
        Self { db_path, backup_dir }
    }

    pub async fn create_backup(&self) -> Result<PathBuf> {
        let timestamp = Utc::now().format("%Y%m%d_%H%M%S");
        let backup_name = format!("crawlerx_backup_{}.db", timestamp);
        let backup_path = self.backup_dir.join(backup_name);

        // 创建备份目录
        std::fs::create_dir_all(&self.backup_dir)?;

        // 复制数据库文件
        std::fs::copy(&self.db_path, &backup_path)?;

        Ok(backup_path)
    }

    pub async fn restore_backup(&self, backup_path: &PathBuf) -> Result<()> {
        // 关闭数据库连接
        // ...

        // 恢复备份文件
        std::fs::copy(backup_path, &self.db_path)?;

        // 重新连接数据库
        // ...

        Ok(())
    }
}
```

### 5.2 自动备份策略

| 策略 | 频率 | 保留数量 |
|------|------|----------|
| 增量备份 | 每小时 | 24 个 |
| 完整备份 | 每天 | 7 天 |
| 压缩备份 | 每周 | 4 周 |

---

## 6. 性能优化

### 6.1 SQLite 优化设置

```sql
-- WAL 模式 (更好的并发)
PRAGMA journal_mode=WAL;

-- 同步设置 (性能与安全的平衡)
PRAGMA synchronous=NORMAL;

-- 缓存大小 (64MB)
PRAGMA cache_size=-64000;

-- 临时存储在内存
PRAGMA temp_store=MEMORY;

-- 页面大小
PRAGMA page_size=4096;

-- mmap 大小
PRAGMA mmap_size=30000000000;
```

### 6.2 查询优化

```sql
-- 使用索引
CREATE INDEX idx_tasks_status_created
ON extraction_tasks(status, created_at DESC);

-- 覆盖索引
CREATE INDEX idx_tasks_list
ON extraction_tasks(status, name, created_at);

-- 分页查询
SELECT * FROM extraction_tasks
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

---

## 7. 数据迁移

### 7.1 版本迁移

```rust
// src-tauri/src/repository/migrate.rs
pub async fn migrate_database(pool: &SqlitePool, from_version: i32, to_version: i32) -> Result<()> {
    match (from_version, to_version) {
        (1, 2) => {
            // v1 -> v2 迁移
            sqlx::query("ALTER TABLE extraction_tasks ADD COLUMN description TEXT")
                .execute(pool)
                .await?;
        }
        (2, 3) => {
            // v2 -> v3 迁移
            sqlx::query("CREATE INDEX idx_tasks_next_run ON extraction_tasks(next_run_at)")
                .execute(pool)
                .await?;
        }
        _ => {}
    }

    // 更新版本号
    sqlx::query("INSERT OR REPLACE INTO settings (key, value) VALUES ('db_version', ?1)")
        .bind(to_version)
        .execute(pool)
        .await?;

    Ok(())
}
```

---

## 8. 附录

### 8.1 数据库文件位置

| 平台 | 数据库位置 |
|------|------------|
| **Windows** | `%APPDATA%/CrawlerX/data/crawlerx.db` |
| **macOS** | `~/Library/Application Support/CrawlerX/data/crawlerx.db` |
| **Linux** | `~/.config/CrawlerX/data/crawlerx.db` |

### 8.2 数据库大小预估

| 数据类型 | 单条记录大小 | 1000条预估 | 10000条预估 |
|----------|--------------|-------------|--------------|
| 任务 | ~2KB | 2MB | 20MB |
| 目标 | ~1KB | 1MB | 10MB |
| 结果 | ~5KB | 5MB | 50MB |
| **总计** | - | **~8MB** | **~80MB** |

---

**文档变更记录**

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|----------|------|
| v1.0.0 | 2025-12-25 | 初始版本 - SQLite + PostgreSQL 同步架构 | AI System Architect |

---

*本文档定义了 CrawlerX 桌面应用的数据库设计，采用 SQLite 本地存储为主，PostgreSQL 云端同步为辅的混合架构。*
