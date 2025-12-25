# CrawlerX 数据库设计文档 (DDD)
## Database Design Document

> **版本**: v1.0.0
> **日期**: 2025-12-25
> **项目代号**: CrawlerX
> **文档状态**: 正式发布

---

## 📋 文档信息

| 项目 | 内容 |
|------|------|
| **文档名称** | CrawlerX 数据库设计文档 |
| **文档版本** | v2.0.0 (统一PostgreSQL存储) |
| **创建日期** | 2025-12-25 |
| **更新日期** | 2025-12-25 |
| **文档作者** | AI System Architect |
| **审核状态** | 待审核 |

---

## 1. 数据库概述

### 1.1 数据库选型

| 数据库 | 用途 | 版本 | 理由 |
|--------|------|------|------|
| **PostgreSQL** | 统一存储 | 15.x | ACID支持、JSONB、全文搜索、分区表、UNLOGGED表 |

### 1.2 统一存储策略

**设计理念**: 简化架构、降低运维成本、提高数据一致性

| 原存储方案 | PostgreSQL替代方案 | 优势 |
|------------|---------------------|------|
| Redis | UNLOGGED表 + SKIP LOCKED队列 | 单一事务、ACID保证 |
| MinIO | BYTEA字段 + 文件系统路径 | 统一元数据管理 |
| MongoDB | JSONB字段 + 分区表 | SQL查询、联表分析 |

### 1.3 数据库架构

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
│  │                                                             │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │   │
│  │  │ 业务数据表   │ │ JSONB文档表  │ │ UNLOGGED缓存表│     │   │
│  │  │ (持久化存储) │ │ (日志/配置)  │ │ (会话/队列)   │     │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘     │   │
│  │                                                             │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │   │
│  │  │ 文件存储表   │ │ 任务队列表   │ │ 日志分区表   │     │   │
│  │  │ (BYTEA/路径) │ │ (SKIP LOCKED)│ │ (按月分区)   │     │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘     │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    PostgreSQL Extensions                 │   │
│  │  • uuid-ossp • pg_trgm • btree_gin • pg_stat_statements  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 表名 | snake_case, 复数 | `users`, `extraction_tasks` |
| 字段名 | snake_case | `user_id`, `created_at` |
| 索引名 | `idx_表名_字段名` | `idx_users_email` |
| 外键名 | `fk_表名_引用表名` | `fk_tasks_users` |
| 主键名 | `id` (UUID) | - |
| 时间戳 | `created_at`, `updated_at` | - |

---

## 2. PostgreSQL 核心表设计

### 2.1 用户与权限模块

#### 2.1.1 users (用户表)

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100),
    avatar_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'active',
                    -- CHECK (status IN ('active', 'inactive', 'suspended')),

    -- 配额信息
    quota_monthly_extractions INTEGER DEFAULT 1000,
    quota_used_extractions INTEGER DEFAULT 0,
    quota_reset_at TIMESTAMP WITH TIME ZONE,

    -- 时间戳
    last_login_at TIMESTAMP WITH TIME ZONE,
    email_verified_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- 软删除
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- 索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created_at ON users(created_at);

-- 注释
COMMENT ON TABLE users IS '用户表';
COMMENT ON COLUMN users.quota_monthly_extractions IS '月度提取配额';
COMMENT ON COLUMN users.quota_used_extractions IS '已使用提取次数';
```

#### 2.1.2 organizations (组织表)

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    logo_url VARCHAR(500),

    -- 配额信息
    quota_monthly_extractions INTEGER DEFAULT 10000,
    quota_max_members INTEGER DEFAULT 10,

    -- 订阅信息
    subscription_plan VARCHAR(50) DEFAULT 'free',
                        -- CHECK (subscription_plan IN ('free', 'pro', 'enterprise')),
    subscription_expires_at TIMESTAMP WITH TIME ZONE,

    status VARCHAR(20) DEFAULT 'active',
                    -- CHECK (status IN ('active', 'inactive', 'suspended')),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_organizations_slug ON organizations(slug);
CREATE INDEX idx_organizations_status ON organizations(status);

COMMENT ON TABLE organizations IS '组织表';
COMMENT ON COLUMN organizations.subscription_plan IS '订阅计划: free/pro/enterprise';
```

#### 2.1.3 organization_members (组织成员表)

```sql
CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    role VARCHAR(20) DEFAULT 'member',
                -- CHECK (role IN ('owner', 'admin', 'member')),

    -- 权限配置
    permissions JSONB DEFAULT '{}',
                    -- {"can_create_tasks": true, "can_delete_tasks": false}

    invited_by UUID REFERENCES users(id),
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_org_members_org_id ON organization_members(organization_id);
CREATE INDEX idx_org_members_user_id ON organization_members(user_id);
CREATE INDEX idx_org_members_role ON organization_members(role);

COMMENT ON TABLE organization_members IS '组织成员关系表';
COMMENT ON COLUMN organization_members.role IS '角色: owner/admin/member';
```

#### 2.1.4 api_keys (API密钥表)

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,

    name VARCHAR(100) NOT NULL,
    key_hash VARCHAR(255) NOT NULL,  -- SHA-256 hash
    key_prefix VARCHAR(10) NOT NULL,  -- 用于展示前缀，如 "crw_xxx"

    -- 权限范围
    scopes TEXT[] DEFAULT ARRAY['read:tasks', 'write:tasks'],
            -- ARRAY['read:tasks', 'write:tasks', 'read:data', ...]

    -- 限制
    rate_limit_per_minute INTEGER DEFAULT 60,
    ip_whitelist INET[],

    -- 过期时间
    expires_at TIMESTAMP WITH TIME ZONE,
    last_used_at TIMESTAMP WITH TIME ZONE,

    status VARCHAR(20) DEFAULT 'active',
                    -- CHECK (status IN ('active', 'revoked', 'expired')),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_api_keys_org_id ON api_keys(organization_id);
CREATE INDEX idx_api_keys_key_prefix ON api_keys(key_prefix);

COMMENT ON TABLE api_keys IS 'API密钥表';
COMMENT ON COLUMN api_keys.key_prefix IS '密钥前缀，用于标识展示';
```

### 2.2 任务管理模块

#### 2.2.1 extraction_tasks (提取任务表)

```sql
CREATE TABLE extraction_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_number VARCHAR(50) UNIQUE NOT NULL,  -- TASK-20251225-0001

    -- 关联信息
    user_id UUID NOT NULL REFERENCES users(id),
    organization_id UUID REFERENCES organizations(id),
    parent_task_id UUID REFERENCES extraction_tasks(id),

    -- 任务信息
    name VARCHAR(255) NOT NULL,
    description TEXT,
    task_type VARCHAR(20) NOT NULL,
                        -- CHECK (task_type IN ('single', 'batch', 'scheduled', 'recurring')),

    -- 配置 (JSONB)
    config JSONB NOT NULL DEFAULT '{}',
            -- {
            --   "urls": ["url1", "url2"],
            --   "markers": [...],
            --   "options": {...}
            -- }

    -- 调度配置
    schedule_cron VARCHAR(100),
    schedule_timezone VARCHAR(50) DEFAULT 'UTC',
    next_run_at TIMESTAMP WITH TIME ZONE,

    -- 任务状态
    status VARCHAR(20) DEFAULT 'pending',
                    -- CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),

    -- 执行统计
    total_targets INTEGER DEFAULT 0,
    completed_targets INTEGER DEFAULT 0,
    failed_targets INTEGER DEFAULT 0,

    -- 时间信息
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_tasks_user_id ON extraction_tasks(user_id);
CREATE INDEX idx_tasks_org_id ON extraction_tasks(organization_id);
CREATE INDEX idx_tasks_status ON extraction_tasks(status);
CREATE INDEX idx_tasks_task_type ON extraction_tasks(task_type);
CREATE INDEX idx_tasks_next_run_at ON extraction_tasks(next_run_at);
CREATE INDEX idx_tasks_created_at ON extraction_tasks(created_at);

COMMENT ON TABLE extraction_tasks IS '数据提取任务表';
COMMENT ON COLUMN extraction_tasks.config IS '任务配置，JSONB格式';
COMMENT ON COLUMN extraction_tasks.schedule_cron IS 'Cron表达式，用于定时任务';
```

#### 2.2.2 task_targets (任务目标表)

```sql
CREATE TABLE task_targets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES extraction_tasks(id) ON DELETE CASCADE,

    -- 目标信息
    url VARCHAR(2000) NOT NULL,
    url_hash VARCHAR(64) NOT NULL,  -- SHA-256 for deduplication

    -- 截图信息
    screenshot_path VARCHAR(500),  -- MinIO path
    screenshot_width INTEGER,
    screenshot_height INTEGER,

    -- 标记信息 (JSONB)
    markers JSONB NOT NULL DEFAULT '[]',
            -- [
            --   {"id": "m1", "x": 100, "y": 200, "width": 50, "height": 30, "label": "price"}
            -- ]

    -- 目标状态
    status VARCHAR(20) DEFAULT 'pending',
                    -- CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'skipped')),

    -- 结果关联
    result_id UUID REFERENCES extraction_results(id),

    -- 错误信息
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    last_retry_at TIMESTAMP WITH TIME ZONE,

    -- 时间信息
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_targets_task_id ON task_targets(task_id);
CREATE INDEX idx_targets_url_hash ON task_targets(url_hash);
CREATE INDEX idx_targets_status ON task_targets(status);
CREATE INDEX idx_targets_result_id ON task_targets(result_id);

COMMENT ON TABLE task_targets IS '任务目标表，单个任务可包含多个目标';
COMMENT ON COLUMN task_targets.markers IS '标记信息，JSONB数组';
```

### 2.3 提取结果模块

#### 2.3.1 extraction_results (提取结果表)

```sql
CREATE TABLE extraction_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    result_number VARCHAR(50) UNIQUE NOT NULL,  -- RES-20251225-0001

    -- 关联信息
    task_id UUID NOT NULL REFERENCES extraction_tasks(id),
    target_id UUID NOT NULL REFERENCES task_targets(id),
    user_id UUID NOT NULL REFERENCES users(id),

    -- 源信息
    source_url VARCHAR(2000) NOT NULL,

    -- 提取元数据
    extraction_method VARCHAR(20) NOT NULL,
                              -- CHECK (extraction_method IN ('html', 'api', 'hybrid', 'manual')),

    -- 提取数据 (JSONB)
    data JSONB NOT NULL DEFAULT '{}',
         -- {
         --   "fields": [
         --     {"name": "price", "value": "$99.99", "selector": "...", "confidence": 0.95}
         --   ],
         --   "apis": [...]
         -- }

    -- 原始数据快照
    html_snapshot_path VARCHAR(500),
    api_snapshots JSONB DEFAULT '[]',

    -- 质量指标
    confidence_score DECIMAL(3,2),  -- 0.00-1.00
    validation_status VARCHAR(20) DEFAULT 'unknown',
                             -- CHECK (validation_status IN ('valid', 'invalid', 'unknown')),

    -- 性能指标
    extraction_duration_ms INTEGER,

    -- 版本控制
    version INTEGER DEFAULT 1,
    is_latest BOOLEAN DEFAULT true,

    -- 时间信息
    extracted_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(target_id, version)
);

CREATE INDEX idx_results_task_id ON extraction_results(task_id);
CREATE INDEX idx_results_target_id ON extraction_results(target_id);
CREATE INDEX idx_results_user_id ON extraction_results(user_id);
CREATE INDEX idx_results_extracted_at ON extraction_results(extracted_at);
CREATE INDEX idx_results_is_latest ON extraction_results(is_latest);

COMMENT ON TABLE extraction_results IS '数据提取结果表';
COMMENT ON COLUMN extraction_results.data IS '提取的结构化数据，JSONB格式';
COMMENT ON COLUMN extraction_results.confidence_score IS '置信度分数，0-1之间';
```

#### 2.3.2 dom_selectors (DOM选择器表)

```sql
CREATE TABLE dom_selectors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 关联信息
    result_id UUID NOT NULL REFERENCES extraction_results(id),
    target_id UUID NOT NULL REFERENCES task_targets(id),

    -- 选择器信息
    selector_type VARCHAR(20) NOT NULL,
                         -- CHECK (selector_type IN ('xpath', 'css', 'text', 'coordinate')),

    selector_value TEXT NOT NULL,

    -- 字段信息
    field_name VARCHAR(100),
    field_label VARCHAR(100),

    -- 匹配信息
    matched_element_tag VARCHAR(50),
    matched_element_text TEXT,
    matched_element_attributes JSONB DEFAULT '{}',

    -- 可靠性指标
    stability_score DECIMAL(3,2),
    usage_count INTEGER DEFAULT 1,

    -- 模板关联
    is_template BOOLEAN DEFAULT false,
    template_id UUID REFERENCES dom_selectors(id),

    -- 时间信息
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_selectors_result_id ON dom_selectors(result_id);
CREATE INDEX idx_selectors_target_id ON dom_selectors(target_id);
CREATE INDEX idx_selectors_template_id ON dom_selectors(template_id);
CREATE INDEX idx_selectors_is_template ON dom_selectors(is_template);

COMMENT ON TABLE dom_selectors IS 'DOM选择器表，存储字段与选择器的映射';
```

#### 2.3.3 api_mappings (API映射表)

```sql
CREATE TABLE api_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 关联信息
    result_id UUID NOT NULL REFERENCES extraction_results(id),
    target_id UUID NOT NULL REFERENCES task_targets(id),

    -- API信息
    api_url VARCHAR(2000) NOT NULL,
    api_method VARCHAR(10) NOT NULL,
                       -- CHECK (api_method IN ('GET', 'POST', 'PUT', 'DELETE')),

    -- 请求信息
    request_headers JSONB DEFAULT '{}',
    request_query JSONB DEFAULT '{}',
    request_body JSONB DEFAULT '{}',

    -- 响应信息
    response_status INTEGER,
    response_headers JSONB DEFAULT '{}',
    response_body JSONB DEFAULT '{}',

    -- 字段映射
    field_mappings JSONB DEFAULT '{}',
                     -- {"price": "$.data.price", "name": "$.data.productName"}

    -- 动态参数分析
    dynamic_parameters JSONB DEFAULT '{}',
                          -- {"timestamp": {"pattern": ".*", "generation": "current_time"}}

    -- 时间信息
    captured_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_api_mappings_result_id ON api_mappings(result_id);
CREATE INDEX idx_api_mappings_target_id ON api_mappings(target_id);
CREATE INDEX idx_api_mappings_api_url ON api_mappings(api_url);

COMMENT ON TABLE api_mappings IS 'API接口映射表';
```

### 2.4 模板与配置模块

#### 2.4.1 marker_templates (标记模板表)

```sql
CREATE TABLE marker_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 所属信息
    user_id UUID NOT NULL REFERENCES users(id),
    organization_id UUID REFERENCES organizations(id),

    -- 模板信息
    name VARCHAR(100) NOT NULL,
    description TEXT,
    category VARCHAR(50),  -- ecommerce, finance, news, etc.

    -- 适用范围
    url_pattern VARCHAR(500),  -- 支持通配符，如 https://example.com/product/*
    domain_pattern VARCHAR(500),

    -- 标记配置 (JSONB)
    markers JSONB NOT NULL DEFAULT '[]',
            -- [
            --   {"id": "price", "selector": "...", "label": "价格", "type": "text"}
            -- ]

    -- 统计信息
    usage_count INTEGER DEFAULT 0,
    success_rate DECIMAL(5,2),

    -- 共享设置
    is_public BOOLEAN DEFAULT false,
    is_system BOOLEAN DEFAULT false,

    -- 状态
    status VARCHAR(20) DEFAULT 'active',
                    -- CHECK (status IN ('active', 'inactive', 'deprecated')),

    -- 时间信息
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_templates_user_id ON marker_templates(user_id);
CREATE INDEX idx_templates_org_id ON marker_templates(organization_id);
CREATE INDEX idx_templates_category ON marker_templates(category);
CREATE INDEX idx_templates_domain_pattern ON marker_templates(domain_pattern);

COMMENT ON TABLE marker_templates IS '标记模板表';
```

#### 2.4.2 extraction_rules (提取规则表)

```sql
CREATE TABLE extraction_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 所属信息
    user_id UUID REFERENCES users(id),
    organization_id UUID REFERENCES organizations(id),

    -- 规则信息
    name VARCHAR(100) NOT NULL,
    rule_type VARCHAR(50) NOT NULL,
                       -- CHECK (rule_type IN ('validation', 'transformation', 'enrichment')),

    -- 规则配置 (JSONB)
    config JSONB NOT NULL DEFAULT '{}',
            -- {
            --   "field": "price",
            --   "rule": "regex",
            --   "pattern": "^\\$[0-9]+\\.[0-9]{2}$"
            -- }

    -- 应用范围
    apply_to_templates UUID[] DEFAULT ARRAY[]::UUID[],

    -- 优先级
    priority INTEGER DEFAULT 0,

    -- 状态
    is_enabled BOOLEAN DEFAULT true,

    -- 时间信息
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_rules_user_id ON extraction_rules(user_id);
CREATE INDEX idx_rules_org_id ON extraction_rules(organization_id);
CREATE INDEX idx_rules_type ON extraction_rules(rule_type);

COMMENT ON TABLE extraction_rules IS '数据提取规则表';
```

### 2.5 日志与审计模块

#### 2.5.1 audit_logs (审计日志表)

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 操作信息
    actor_type VARCHAR(20) NOT NULL,
                       -- CHECK (actor_type IN ('user', 'system', 'api')),
    actor_id UUID,
    actor_ip INET,

    -- 操作详情
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID,

    -- 变更记录 (JSONB)
    changes JSONB DEFAULT '{}',
            -- {"before": {...}, "after": {...}}

    -- 结果
    status VARCHAR(20) DEFAULT 'success',
                    -- CHECK (status IN ('success', 'failure')),

    -- 额外信息
    metadata JSONB DEFAULT '{}',
    user_agent TEXT,

    -- 时间信息
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 分区表（按月分区）
CREATE TABLE audit_logs_partitioned (
    LIKE audit_logs INCLUDING ALL
) PARTITION BY RANGE (created_at);

-- 创建分区
CREATE TABLE audit_logs_2025_01 PARTITION OF audit_logs_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE INDEX idx_audit_logs_actor ON audit_logs(actor_type, actor_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);

COMMENT ON TABLE audit_logs IS '审计日志表';
```

### 2.6 系统配置模块

#### 2.6.1 system_settings (系统配置表)

```sql
CREATE TABLE system_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 配置键
    key VARCHAR(100) UNIQUE NOT NULL,

    -- 配置值
    value JSONB NOT NULL,

    -- 配置元信息
    description TEXT,
    category VARCHAR(50),
    is_public BOOLEAN DEFAULT false,  -- 是否可被前端读取

    -- 验证规则
    validation_rule TEXT,

    -- 时间信息
    updated_by UUID REFERENCES users(id),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_settings_key ON system_settings(key);
CREATE INDEX idx_settings_category ON system_settings(category);

COMMENT ON TABLE system_settings IS '系统配置表';

-- 初始配置
INSERT INTO system_settings (key, value, description, category) VALUES
    ('max.concurrent_tasks', '100', '最大并发任务数', 'system'),
    ('max.task_duration', '3600', '单个任务最大执行时长(秒)', 'system'),
    ('default.retry_count', '3', '默认重试次数', 'task'),
    ('screenshot.max_size', '10485760', '截图最大大小(字节)', 'upload');
```

---

## 3. PostgreSQL 缓存与会话表 (替代 Redis)

### 3.1 sessions (会话表)

```sql
-- 使用 UNLOGGED 提升性能，可接受会话丢失
CREATE UNLOGGED TABLE sessions (
    id VARCHAR(64) PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- 索引
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
CREATE INDEX idx_sessions_user_id ON sessions(user_id);

-- 自动清理 (使用 pg_cron 或应用层定时任务)
DELETE FROM sessions WHERE expires_at < CURRENT_TIMESTAMP;

COMMENT ON TABLE sessions IS '用户会话表 (UNLOGGED，高性能)';
```

### 3.2 cache (缓存表)

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

-- TTL 清理
DELETE FROM cache WHERE expires_at < CURRENT_TIMESTAMP;

COMMENT ON TABLE cache IS '通用缓存表 (UNLOGGED)';
```

### 3.3 task_queue (任务队列表)

```sql
-- 任务队列表 (使用 SKIP LOCKED 实现队列)
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
    processed_at TIMESTAMP WITH TIME ZONE,
    worker_id VARCHAR(100),
    error_message TEXT
);

-- 索引
CREATE INDEX idx_queue_status ON task_queue(status, priority, created_at);
CREATE INDEX idx_queue_reserved ON task_queue(reserved_at) WHERE status = 'processing';
CREATE INDEX idx_queue_name ON task_queue(queue_name);

-- 出队操作 (FOR UPDATE SKIP LOCKED)
UPDATE task_queue
SET status = 'processing',
    reserved_at = CURRENT_TIMESTAMP,
    attempts = attempts + 1,
    worker_id = 'worker-' || :worker_id
WHERE id = (
    SELECT id FROM task_queue
    WHERE status = 'pending'
      AND queue_name = :queue_name
      AND (attempts < max_attempts OR max_attempts = 0)
    ORDER BY priority DESC, created_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
RETURNING *;

-- 入队操作
INSERT INTO task_queue (queue_name, payload, priority)
VALUES ('extraction', :payload_json, :priority);

-- 完成任务
UPDATE task_queue
SET status = :status,
    processed_at = CURRENT_TIMESTAMP
WHERE id = :task_id;

-- 失败重试
UPDATE task_queue
SET status = 'pending',
    error_message = :error_msg
WHERE id = :task_id
  AND attempts < max_attempts;

COMMENT ON TABLE task_queue IS '任务队列表 (使用 SKIP LOCKED 实现并发安全)';
```

### 3.4 distributed_locks (分布式锁表)

```sql
-- 分布式锁表
CREATE TABLE distributed_locks (
    lock_key VARCHAR(255) PRIMARY KEY,
    locked_by VARCHAR(100) NOT NULL,
    locked_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- 索引
CREATE INDEX idx_locks_expires_at ON distributed_locks(expires_at);

-- 获取锁 (INSERT ... ON CONFLICT)
INSERT INTO distributed_locks (lock_key, locked_by, expires_at)
VALUES (:lock_key, :worker_id, CURRENT_TIMESTAMP + INTERVAL '5 minutes')
ON CONFLICT (lock_key) DO UPDATE
SET locked_by = EXCLUDED.locked_by,
    expires_at = EXCLUDED.expires_at
WHERE distributed_locks.expires_at < CURRENT_TIMESTAMP
RETURNING lock_key;

-- 释放锁
DELETE FROM distributed_locks
WHERE lock_key = :lock_key
  AND locked_by = :worker_id;

-- 自动清理过期锁
DELETE FROM distributed_locks WHERE expires_at < CURRENT_TIMESTAMP;

COMMENT ON TABLE distributed_locks IS '分布式锁表';
```

### 3.5 counters (计数器表)

```sql
-- 计数器表
CREATE UNLOGGED TABLE counters (
    key VARCHAR(255) PRIMARY KEY,
    value BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 原子递增
INSERT INTO counters (key, value)
VALUES (:key, 1)
ON CONFLICT (key) DO UPDATE
SET value = counters.value + 1,
    updated_at = CURRENT_TIMESTAMP
RETURNING value;

-- 重置计数器
UPDATE counters
SET value = 0, updated_at = CURRENT_TIMESTAMP
WHERE key = :key;

COMMENT ON TABLE counters IS '通用计数器表 (UNLOGGED)';
```

### 3.6 rate_limits (限流表)

```sql
-- 限流表
CREATE UNLOGGED TABLE rate_limits (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL,
    window_start TIMESTAMP WITH TIME ZONE NOT NULL,
    request_count INTEGER DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- 唯一约束确保同一窗口期只有一条记录
    UNIQUE(user_id, window_start)
);

-- 索引
CREATE INDEX idx_rate_limits_user_window ON rate_limits(user_id, window_start);

-- 检查并递增
WITH window AS (
    SELECT date_trunc('minute', CURRENT_TIMESTAMP) AS window_start
)
INSERT INTO rate_limits (user_id, window_start)
SELECT :user_id, window_start FROM window
ON CONFLICT (user_id, window_start) DO UPDATE
SET request_count = rate_limits.request_count + 1
RETURNING request_count <= 60 AS allowed; -- 每分钟60次

-- 自动清理过期窗口
DELETE FROM rate_limits
WHERE window_start < date_trunc('minute', CURRENT_TIMESTAMP) - INTERVAL '1 hour';

COMMENT ON TABLE rate_limits IS 'API限流表 (UNLOGGED)';
```

---

## 4. PostgreSQL 文件存储表 (替代 MinIO)

### 4.1 stored_files (文件存储表)

```sql
-- 文件存储表 (混合存储: 小文件存数据库，大文件存文件系统)
CREATE TABLE stored_files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50) NOT NULL,
                    -- CHECK (file_type IN ('screenshot', 'snapshot', 'export', 'backup', 'avatar')),
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(100),

    -- 存储方式
    storage_type VARCHAR(20) NOT NULL,
                       -- CHECK (storage_type IN ('database', 'filesystem')),
    file_path VARCHAR(500),  -- 文件系统路径 (当 storage_type = 'filesystem')
    file_data BYTEA,         -- 文件二进制 (当 storage_type = 'database', 适用于 <1MB 小文件)

    -- 元数据 (JSONB)
    metadata JSONB DEFAULT '{}',
              -- {"width": 1920, "height": 1080, "thumbnail_path": "..."}

    -- 关联信息
    related_object_type VARCHAR(50),  -- 'task', 'target', 'user', etc.
    related_object_id UUID,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 索引
CREATE INDEX idx_files_type ON stored_files(file_type);
CREATE INDEX idx_files_related ON stored_files(related_object_type, related_object_id);
CREATE INDEX idx_files_created ON stored_files(created_at DESC);

-- 示例: 插入小文件 (<1MB)
INSERT INTO stored_files (file_name, file_type, file_size, mime_type, storage_type, file_data, related_object_type, related_object_id)
VALUES ('screenshot.png', 'screenshot', 102400, 'image/png', 'database', '\x...'::BYTEA, 'target', 'uuid');

-- 示例: 插入大文件 (≥1MB)
INSERT INTO stored_files (file_name, file_type, file_size, mime_type, storage_type, file_path, related_object_type, related_object_id)
VALUES ('snapshot.png', 'snapshot', 5242880, 'image/png', 'filesystem', '/data/crawlerx/screenshots/2025/01/uuid.png', 'target', 'uuid');

COMMENT ON TABLE stored_files IS '文件存储表 (小文件存数据库，大文件存文件系统)';
COMMENT ON COLUMN stored_files.storage_type IS 'database: <1MB文件存BYTEA; filesystem: ≥1MB文件存路径';
```

### 4.2 文件系统目录结构

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
├── exports/
│   └── {year}/
│       └── {month}/
│           └── {task_id}_{format}.{ext}
└── backups/
    └── crawlerx_{date}.dump
```

---

## 5. PostgreSQL 日志表 (替代 MongoDB)

### 5.1 operation_logs (操作日志表)

```sql
-- 操作日志表 (分区表，按月分区)
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

    -- 上下文 (JSONB)
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

-- 索引
CREATE INDEX idx_op_logs_user ON operation_logs(user_id, created_at DESC);
CREATE INDEX idx_op_logs_action ON operation_logs(action);
CREATE INDEX idx_op_logs_resource ON operation_logs(resource_type, resource_id);
CREATE INDEX idx_op_logs_request ON operation_logs USING GIN (request);
CREATE INDEX idx_op_logs_context ON operation_logs USING GIN (context);

-- TTL 清理 (保留90天)
DELETE FROM operation_logs WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '90 days';

COMMENT ON TABLE operation_logs IS '操作日志表 (分区表，JSONB存储)';
```

### 5.2 error_logs (错误日志表)

```sql
-- 错误日志表 (分区表)
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

-- 分区 (按月)
CREATE TABLE error_logs_partitioned (
    LIKE error_logs INCLUDING ALL
) PARTITION BY RANGE (created_at);

-- 创建分区
CREATE TABLE error_logs_2025_01 PARTITION OF error_logs_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- 索引
CREATE INDEX idx_error_logs_task ON error_logs(task_id);
CREATE INDEX idx_error_logs_type ON error_logs(error_type);
CREATE INDEX idx_error_logs_resolved ON error_logs(is_resolved);
CREATE INDEX idx_error_logs_context ON error_logs USING GIN (context);

-- TTL 清理 (保留30天)
DELETE FROM error_logs WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '30 days';

COMMENT ON TABLE error_logs IS '错误日志表 (分区表，JSONB存储)';
```

### 5.3 performance_metrics (性能指标表)

```sql
-- 性能指标表 (时序数据，可选 TimescaleDB 扩展)
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

-- TTL 清理 (保留7天)
DELETE FROM performance_metrics WHERE timestamp < CURRENT_TIMESTAMP - INTERVAL '7 days';

COMMENT ON TABLE performance_metrics IS '性能指标表 (分区表，JSONB存储)';
```

---

## 6. 数据库视图设计

### 6.1 任务统计视图

```sql
CREATE VIEW task_statistics AS
SELECT
    t.id as task_id,
    t.name as task_name,
    t.status,
    COUNT(tr.id) as total_targets,
    SUM(CASE WHEN tr.status = 'completed' THEN 1 ELSE 0 END) as completed_count,
    SUM(CASE WHEN tr.status = 'failed' THEN 1 ELSE 0 END) as failed_count,
    SUM(CASE WHEN tr.status = 'pending' THEN 1 ELSE 0 END) as pending_count,
    ROUND(
        SUM(CASE WHEN tr.status = 'completed' THEN 1 ELSE 0 END)::DECIMAL /
        NULLIF(COUNT(tr.id), 0) * 100, 2
    ) as completion_rate,
    t.created_at,
    t.started_at,
    t.completed_at
FROM extraction_tasks t
LEFT JOIN task_targets tr ON tr.task_id = t.id
GROUP BY t.id;

COMMENT ON VIEW task_statistics IS '任务统计视图';
```

### 6.2 用户使用统计视图

```sql
CREATE VIEW user_usage_statistics AS
SELECT
    u.id as user_id,
    u.username,
    u.email,
    u.quota_monthly_extractions,
    u.quota_used_extractions,
    ROUND(
        u.quota_used_extractions::DECIMAL /
        NULLIF(u.quota_monthly_extractions, 0) * 100, 2
    ) as quota_usage_percent,
    COUNT(DISTINCT t.id) as total_tasks,
    COUNT(DISTINCT CASE WHEN t.status = 'completed' THEN t.id END) as completed_tasks,
    SUM(tr.completed_targets) as total_extractions
FROM users u
LEFT JOIN extraction_tasks t ON t.user_id = u.id
LEFT JOIN task_targets tr ON tr.task_id = t.id
GROUP BY u.id;

COMMENT ON VIEW user_usage_statistics IS '用户使用统计视图';
```

---

## 7. 数据迁移脚本

### 7.1 初始化脚本

```sql
-- init_database.sql

-- 创建扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- 模糊搜索
CREATE EXTENSION IF NOT EXISTS "btree_gin"; -- 复合索引

-- 创建函数: 自动更新 updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 创建触发器
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_tasks_updated_at BEFORE UPDATE ON extraction_tasks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ... 其他表的触发器
```

### 7.2 数据清理脚本

```sql
-- 清理过期数据

-- 清理90天前的审计日志
DELETE FROM audit_logs
WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '90 days';

-- 清理软删除的数据（30天前）
DELETE FROM extraction_tasks
WHERE deleted_at < CURRENT_TIMESTAMP - INTERVAL '30 days';

DELETE FROM task_targets
WHERE deleted_at < CURRENT_TIMESTAMP - INTERVAL '30 days';

-- 清理旧的提取结果（保留最近10个版本）
DELETE FROM extraction_results
WHERE (target_id, version) NOT IN (
    SELECT target_id, version
    FROM (
        SELECT target_id, version
        FROM extraction_results
        ORDER BY created_at DESC
        LIMIT 10
    ) t
);
```

---

## 8. 数据库监控

### 8.1 监控指标

| 指标 | 查询 | 告警阈值 |
|------|------|----------|
| 连接数 | `SELECT count(*) FROM pg_stat_activity` | > 80% max_connections |
| 慢查询 | `pg_stat_statements` | avg_time > 1s |
| 表膨胀 | `pg_stat_user_tables` | bloat > 20% |
| 索引使用 | `pg_stat_user_indexes` | idx_scan = 0 |
| 缓存命中率 | `blks_hit / (blks_hit + blks_read)` | < 95% |

### 8.2 定期维护

```sql
-- 定期VACUUM ANALYZE
VACUUM ANALYZE extraction_tasks;
VACUUM ANALYZE task_targets;
VACUUM ANALYZE extraction_results;

-- 重建索引
REINDEX TABLE CONCURRENTLY extraction_tasks;
REINDEX TABLE CONCURRENTLY extraction_results;

-- 更新统计信息
ANALYZE extraction_tasks;
ANALYZE task_targets;
ANALYZE extraction_results;
```

---

## 9. 备份与恢复

### 9.1 备份策略

| 类型 | 频率 | 保留期 | 存储 |
|------|------|--------|------|
| 全量备份 | 每日 | 30天 | MinIO |
| 增量备份 | 每小时 | 7天 | MinIO |
| WAL归档 | 实时 | 3天 | MinIO |

### 9.2 备份脚本

```bash
#!/bin/bash
# backup.sh

# 全量备份
pg_dump -h localhost -U crawlerx -d crawlerx \
  --format=custom \
  --file=/backup/crawlerx_$(date +%Y%m%d).dump

# 上传到MinIO
mc cp /backup/crawlerx_*.dump minio/crawlerx-backups/

# 清理旧备份（保留30天）
find /backup -name "crawlerx_*.dump" -mtime +30 -delete
```

### 9.3 恢复脚本

```bash
#!/bin/bash
# restore.sh

# 从MinIO下载
mc cp minio/crawlerx-backups/crawlerx_20251225.dump /backup/

# 恢复数据库
pg_restore -h localhost -U crawlerx -d crawlerx_new \
  --clean --if-exists \
  /backup/crawlerx_20251225.dump
```

---

## 10. 附录

### 10.1 ER图

```
┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
│    users     │────────▶│organization_members│◀────────│organizations│
└──────────────┘         └──────────────────┘         └──────────────┘
        │                                                         │
        │                                                         │
        ▼                                                         │
┌──────────────┐         ┌──────────────────┐                    │
│    api_keys  │         │extraction_tasks  │◀───────────────────┘
└──────────────┘         └──────────────────┘
                                 │
                                 │
                                 ▼
                        ┌──────────────────┐         ┌──────────────┐
                        │  task_targets    │────────▶│extraction_  │
                        └──────────────────┘         │  results    │
                                 │                    └──────────────┘
                                 │                             │
                                 ▼                             ▼
                        ┌──────────────────┐         ┌──────────────┐
                        │ dom_selectors    │         │api_mappings │
                        └──────────────────┘         └──────────────┘
```

### 10.2 参考文档

| 文档 | 链接 |
|------|------|
| 系统需求文档 | `./01-SRD_System_Requirements_Document.md` |
| 系统架构设计文档 | `./02-SAD_System_Architecture_Design.md` |
| 任务规划文档 | `./04-TPD_Task_Planning_Document.md` |

---

**文档变更记录**

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|----------|------|
| v1.0.0 | 2025-12-25 | 初始版本 | AI System Architect |
| v2.0.0 | 2025-12-25 | 统一PostgreSQL存储策略，移除Redis/MinIO/MongoDB依赖 | AI System Architect |

---

*本文档定义了 CrawlerX 系统的完整数据库设计，统一使用 PostgreSQL 进行数据存储，包括业务数据、缓存、队列、文件、日志等所有数据类型。*

---

## 附录: PostgreSQL 扩展推荐

```sql
-- 必需扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";           -- UUID 生成
CREATE EXTENSION IF NOT EXISTS "pg_trgm";            -- 模糊搜索
CREATE EXTENSION IF NOT EXISTS "btree_gin";          -- 复合索引
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements"; -- 查询性能分析

-- 可选扩展 (按需启用)
-- CREATE EXTENSION IF NOT EXISTS "pg_cron";         -- 定时任务
-- CREATE EXTENSION IF NOT EXISTS "timescaledb";     -- 时序数据
-- CREATE EXTENSION IF NOT EXISTS "pg_partman";      -- 自动分区管理
```

## 附录: 性能优化建议

```sql
-- 配置建议 (postgresql.conf)

# 内存配置
shared_buffers = 4GB              -- 总内存的 25%
effective_cache_size = 12GB       -- 总内存的 75%
work_mem = 64MB                   -- 单个操作内存

# WAL 配置
wal_buffers = 16MB
min_wal_size = 2GB
max_wal_size = 8GB
wal_compression = on

# 查询优化 (SSD)
random_page_cost = 1.1
effective_io_concurrency = 200

# 连接配置
max_connections = 200
max_worker_processes = 16
```
