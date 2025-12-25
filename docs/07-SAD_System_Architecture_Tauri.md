# CrawlerX 系统架构设计文档 (Tauri 2.0 + Vue 3.0)
## System Architecture Design for Tauri 2.0

> **版本**: v1.1.0
> **日期**: 2025-12-25
> **项目代号**: CrawlerX
> **架构类型**: CS 桌面应用 (Tauri 2.0 + Vue 3.0)

---

## 📋 文档信息

| 项目 | 内容 |
|------|------|
| **文档名称** | CrawlerX 系统架构设计文档 (Tauri 2.0 + Vue 3.0) |
| **文档版本** | v1.1.0 |
| **创建日期** | 2025-12-25 |
| **架构类型** | Tauri 2.0 CS 桌面应用 |

---

## 1. 架构概述

### 1.1 系统定位

CrawlerX 是一个基于 **Tauri 2.0** 的跨平台桌面应用，采用 Rust 后端 + Vue 3 前端的混合架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                     CrawlerX Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────┐    IPC     ┌─────────┐    Native     ┌─────────┐ │
│    │ WebView │◀─────────▶│ Rust Core│◀─────────────▶│ OS APIs │ │
│    │(Vue 3)  │           │(Tauri)   │               │(Files/  │ │
│    └─────────┘           └─────────┘               │ Network)│ │
│                                                           │   │
│                                                           └───┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 架构对比

| 特性 | Web 应用 (原方案) | Tauri 桌面应用 (新方案) |
|------|-------------------|-------------------------|
| **部署** | 服务器托管 | 本地安装 |
| **数据存储** | 远程数据库 | 本地 SQLite + 可选云端同步 |
| **离线能力** | 有限 | 完全离线可用 |
| **系统集成** | Web API | 原生 OS 集成 |
| **安装包大小** | - | ~10 MB |
| **启动速度** | 网络依赖 | 即时启动 |

### 1.3 设计原则

| 原则 | 说明 |
|------|------|
| **本地优先** | 核心功能本地运行，离线可用 |
| **渐进增强** | 基础功能本地，高级功能可选云端 |
| **数据主权** | 用户数据存储在本地，用户完全控制 |
| **跨平台** | 一套代码，支持 Windows/macOS/Linux |
| **原生体验** | 使用系统原生组件和 API |

---

## 2. 总体架构

### 2.1 逻辑架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CrawlerX 逻辑架构 (Tauri + Vue)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     前端层 (WebView - Vue 3)                         │   │
│  │                                                                     │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  页面组件     │  │  状态管理     │  │  IPC 通信     │              │   │
│  │  │  (SFC Views)  │  │  (Pinia)     │  │  (Tauri API)  │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                                    IPC Bridge                              │
│                                      │                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     后端层 (Rust Core - Tauri)                       │   │
│  │                                                                     │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  Command     │  │  Service     │  │  Repository  │              │   │
│  │  │  Handler     │  │  Layer       │  │  Layer       │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │                                                                     │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  Vision      │  │  Data        │  │  Task        │              │   │
│  │  │  Service     │  │  Service     │  │  Scheduler   │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        存储层                                       │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    SQLite (本地主存储)                        │   │   │
│  │  │  • 用户配置  • 任务数据  • 提取结果  • 缓存数据               │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              ↕ (可选同步)                          │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    PostgreSQL (云端备份/同步)                │   │   │
│  │  │  • 跨设备同步  • 云端备份  • 团队协作                        │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        外部层                                       │   │
│  │                                                                     │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │   │
│  │  │  Browser   │ │  Target    │ │  File      │ │  Cloud     │       │   │
│  │  │  Driver    │ │  Websites  │ │  System    │ │  Service   │       │   │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
crawlerx/
├── src/                          # Vue 3 前端源码
│   ├── components/                # UI 组件
│   │   ├── screenshot/           # 截图标记组件
│   │   │   ├── CanvasMarker.vue
│   │   │   └── ImageUpload.vue
│   │   ├── tasks/                # 任务管理组件
│   │   │   ├── TaskList.vue
│   │   │   ├── TaskCreate.vue
│   │   │   └── TaskDetail.vue
│   │   ├── results/               # 结果展示组件
│   │   │   ├── ResultTable.vue
│   │   │   └── DataExport.vue
│   │   └── common/                # 通用组件
│   ├── views/                     # 页面组件
│   │   ├── Dashboard.vue
│   │   ├── Tasks.vue
│   │   ├── Results.vue
│   │   └── Settings.vue
│   ├── stores/                    # Pinia 状态管理
│   │   ├── task.ts
│   │   ├── ui.ts
│   │   └── user.ts
│   ├── composables/               # Vue Composables
│   │   ├── useTasks.ts
│   │   └── useResults.ts
│   ├── services/                  # API 服务层
│   │   ├── tauri.ts               # Tauri 封装
│   │   ├── task.ts
│   │   └── data.ts
│   ├── utils/                     # 工具函数
│   ├── types/                     # TypeScript 类型定义
│   ├── App.vue
│   └── main.ts
│
├── src-tauri/                     # Rust 后端源码
│   ├── src/
│   │   ├── main.rs                # 程序入口
│   │   ├── lib.rs                 # 库入口
│   │   ├── commands/              # Tauri Commands (API 端点)
│   │   │   ├── mod.rs
│   │   │   ├── vision.rs          # 视觉识别命令
│   │   │   ├── data.rs            # 数据提取命令
│   │   │   ├── task.rs            # 任务管理命令
│   │   │   ├── export.rs          # 数据导出命令
│   │   │   └── settings.rs        # 设置命令
│   │   ├── services/              # 业务服务层
│   │   │   ├── mod.rs
│   │   │   ├── vision_service.rs  # 视觉识别服务
│   │   │   ├── data_service.rs    # 数据提取服务
│   │   │   ├── browser_pool.rs    # 浏览器连接池
│   │   │   └── sync_service.rs    # 云同步服务 (可选)
│   │   ├── repository/            # 数据访问层
│   │   │   ├── mod.rs
│   │   │   ├── sqlite.rs          # SQLite 实现
│   │   │   └── models.rs           # 数据模型
│   │   └── utils/                 # 工具模块
│   │       ├── mod.rs
│   │       ├── image.rs           # 图像处理
│   │       └── crypto.rs          # 加密工具
│   ├── Cargo.toml                 # Rust 依赖配置
│   ├── tauri.conf.json            # Tauri 配置
│   ├── build.rs                   # 构建脚本
│   └── icons/                     # 应用图标
│
├── data/                          # 本地数据目录
│   ├── database.db                # SQLite 数据库文件
│   ├── cache/                     # 缓存目录
│   ├── exports/                   # 导出文件
│   └── logs/                      # 日志文件
│
├── dist/                          # 前端构建产物
├── docs/                          # 项目文档
├── package.json                   # Node.js 依赖
├── vite.config.ts                 # Vite 配置
└── tsconfig.json                  # TypeScript 配置
```

---

## 3. IPC 通信架构

### 3.1 Tauri IPC 机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tauri IPC 通信架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Vue 3 Frontend                          Rust Backend          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                         │   │
│  │  ┌──────────────┐         ┌──────────────┐             │   │
│  │  │  UI Layer    │         │  Tauri API   │             │   │
│  │  │  (SFC Views) │────────▶│  (invoke)    │             │   │
│  │  └──────────────┘         └──────────────┘             │   │
│  │                                │                         │   │
│  │                                ▼                         │   │
│  │                         ┌──────────────┐             │   │
│  │                         │   Commands   │◀─────────────┤   │
│  │                         │  (#[tauri::  │   Rust      │   │
│  │                         │   command])  │   Backend    │   │
│  │                         └──────────────┘             │   │
│  │                                │                         │   │
│  │                                ▼                         │   │
│  │                         ┌──────────────┐             │   │
│  │  ┌──────────────┐         │   Services   │             │   │
│  │  │  Event       │◀────────│   (Business  │             │   │
│  │  │  Listener    │  emit   │    Logic)    │             │   │
│  │  └──────────────┘         └──────────────┘             │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Command 定义

```rust
// Rust 端 - src-tauri/src/commands/task.rs
use serde::{Deserialize, Serialize};
use crate::services::TaskService;

#[derive(Debug, Serialize, Deserialize)]
pub struct CreateTaskRequest {
    pub name: String,
    pub config: TaskConfig,
    pub schedule: Option<String>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct TaskResponse {
    pub id: String,
    pub name: String,
    pub status: String,
    pub created_at: i64,
}

#[tauri::command]
pub async fn create_task(
    req: CreateTaskRequest,
    service: tauri::State<'_, TaskService>,
) -> Result<TaskResponse, String> {
    service.create(req)
        .await
        .map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn get_tasks(
    service: tauri::State<'_, TaskService>,
) -> Result<Vec<TaskResponse>, String> {
    service.list()
        .await
        .map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn delete_task(
    id: String,
    service: tauri::State<'_, TaskService>,
) -> Result<(), String> {
    service.delete(&id)
        .await
        .map_err(|e| e.to_string())
}
```

### 3.3 前端调用

```vue
<!-- Vue 3 端 - src/views/Tasks.vue -->
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { invoke } from '@tauri-apps/api/core'
import { listen } from '@tauri-apps/api/event'

interface Task {
  id: string
  name: string
  status: string
  created_at: number
}

interface CreateTaskRequest {
  name: string
  config: TaskConfig
  schedule?: string
}

const tasks = ref<Task[]>([])
const loading = ref(false)

// 调用 Rust Command
async function fetchTasks() {
  loading.value = true
  try {
    tasks.value = await invoke<Task[]>('get_tasks')
  } finally {
    loading.value = false
  }
}

async function createTask(req: CreateTaskRequest) {
  await invoke<Task>('create_task', { req })
  await fetchTasks()
}

async function deleteTask(id: string) {
  await invoke('delete_task', { id })
  await fetchTasks()
}

// 监听 Rust Event
onMounted(async () => {
  await fetchTasks()

  const unlisten = await listen<TaskProgress>('task-progress', (event) => {
    console.log('Task progress:', event.payload)
  })

  onUnmounted(() => {
    unlisten()
  })
})
</script>

<template>
  <div v-loading="loading">
    <!-- Task list UI -->
  </div>
</template>
```

### 3.4 Event 发送

```rust
// Rust 端发送事件给前端
use tauri::Manager;

fn emit_task_progress(app: &tauri::AppHandle, task_id: &str, progress: u32) {
    app.emit("task-progress", TaskProgressEvent {
        task_id: task_id.to_string(),
        progress,
        status: "running".to_string(),
    }).ok();
}
```

---

## 4. 核心服务设计

### 4.1 Vision Service (视觉识别服务)

```rust
// src-tauri/src/services/vision_service.rs
use image::{DynamicImage, ImageFormat};
use opencv::core::{Mat, Point, Rect, Size};
use opencv::imgproc;

pub struct VisionService {
    browser_pool: Arc<BrowserPool>,
}

impl VisionService {
    pub fn new(browser_pool: Arc<BrowserPool>) -> Self {
        Self { browser_pool }
    }

    pub async fn analyze_screenshot(
        &self,
        url: &str,
        screenshot_data: &[u8],
        markers: &[Marker],
    ) -> Result<Vec<DomMapping>, VisionError> {
        // 1. 加载截图
        let img = image::load_from_memory(screenshot_data)?;
        let (width, height) = img.dimensions();

        // 2. 获取页面 DOM
        let browser = self.browser_pool.acquire().await?;
        browser.goto(url).await?;

        // 3. 获取页面尺寸
        let viewport_size = browser.window_size().await?;
        let scale_x = viewport_size.width as f32 / width as f32;
        let scale_y = viewport_size.height as f32 / height as f32;

        // 4. 映射每个标记
        let mut mappings = Vec::new();
        for marker in markers {
            let dom_x = (marker.x as f32 * scale_x) as u32;
            let dom_y = (marker.y as f32 * scale_y) as u32;

            // 查找 DOM 元素
            let element = browser.find_element(By::Coordinates(dom_x, dom_y)).await?;

            mappings.push(DomMapping {
                marker_id: marker.id.clone(),
                selector: element.selector().await?,
                xpath: element.xpath().await?,
                confidence: calculate_confidence(&marker, &element),
            });
        }

        Ok(mappings)
    }
}
```

### 4.2 Data Service (数据提取服务)

```rust
// src-tauri/src/services/data_service.rs
use thirtyfour::prelude::*;
use tokio::sync::Semaphore;

pub struct DataService {
    driver: WebDriver,
}

impl DataService {
    pub async fn new() -> Result<Self> {
        let caps = DesiredCapabilities::chrome();
        let driver = WebDriver::new("http://localhost:9515", caps).await?;
        Ok(Self { driver })
    }

    pub async fn extract_data(
        &self,
        url: &str,
        mappings: &[DomMapping],
    ) -> Result<Vec<ExtractedField>> {
        self.driver.goto(url).await?;

        let mut results = Vec::new();
        for mapping in mappings {
            let element = self.driver.find(By::XPath(&mapping.xpath)).await?;

            // 提取文本
            let text = element.text().await?;

            // 提取属性
            let mut attributes = std::collections::HashMap::new();
            for attr in &["href", "src", "data-id"] {
                if let Ok(value) = element.attr(attr).await {
                    if let Some(v) = value {
                        attributes.insert(attr.to_string(), v);
                    }
                }
            }

            results.push(ExtractedField {
                name: mapping.marker_id.clone(),
                value: text,
                selector: mapping.selector.clone(),
                attributes,
            });
        }

        Ok(results)
    }
}
```

### 4.3 Task Scheduler (任务调度服务)

```rust
// src-tauri/src/services/task_scheduler.rs
use tokio_cron_scheduler::{Job, JobScheduler, JobSchedulerError};
use std::sync::Arc;
use tokio::sync::Mutex;

pub struct TaskScheduler {
    scheduler: Arc<Mutex<JobScheduler>>,
}

impl TaskScheduler {
    pub async fn new() -> Result<Self, JobSchedulerError> {
        let scheduler = JobScheduler::new().await?;
        Ok(Self {
            scheduler: Arc::new(Mutex::new(scheduler)),
        })
    }

    pub async fn add_scheduled_task(
        &self,
        task_id: String,
        cron_expression: &str,
        executor: Arc<dyn TaskExecutor>,
    ) -> Result<(), JobSchedulerError> {
        let job = Job::new_async(cron_expression, move |_uuid, _l| {
            let task_id = task_id.clone();
            let executor = executor.clone();
            Box::pin(async move {
                executor.execute(&task_id).await;
            })
        })?;

        self.scheduler.lock().await.add(job).await?;
        Ok(())
    }

    pub async fn start(&self) -> Result<(), JobSchedulerError> {
        self.scheduler.lock().await.start().await?;
        Ok(())
    }
}

#[async_trait::async_trait]
pub trait TaskExecutor: Send + Sync {
    async fn execute(&self, task_id: &str);
}
```

---

## 5. 数据存储设计

### 5.1 SQLite 本地数据库

```rust
// src-tauri/src/repository/sqlite.rs
use sqlx::{SqlitePool, Row};
use anyhow::Result;

pub struct SqliteRepository {
    pool: SqlitePool,
}

impl SqliteRepository {
    pub async fn new(db_path: &str) -> Result<Self> {
        // 创建数据目录
        if let Some(parent) = std::path::Path::new(db_path).parent() {
            std::fs::create_dir_all(parent)?;
        }

        let pool = SqlitePool::connect(db_path).await?;

        // 启用 WAL 模式 (更好的并发性能)
        sqlx::query("PRAGMA journal_mode=WAL")
            .execute(&pool)
            .await?;

        // 初始化表结构
        self::migrate::run_migrations(&pool).await?;

        Ok(Self { pool })
    }

    pub async fn create_task(
        &self,
        task: &NewTask,
    ) -> Result<String> {
        let id = Uuid::new_v4().to_string();

        sqlx::query(
            r#"
            INSERT INTO extraction_tasks (id, name, config, status, created_at, updated_at)
            VALUES (?1, ?2, ?3, ?4, ?5, ?6)
            "#
        )
        .bind(&id)
        .bind(&task.name)
        .bind(serde_json::to_string(&task.config)?)
        .bind("pending")
        .bind(Utc::now().timestamp())
        .bind(Utc::now().timestamp())
        .execute(&self.pool)
        .await?;

        Ok(id)
    }

    pub async fn get_tasks(&self) -> Result<Vec<Task>> {
        let tasks = sqlx::query_as::<_, Task>(
            "SELECT * FROM extraction_tasks ORDER BY created_at DESC"
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(tasks)
    }
}
```

### 5.2 数据库表设计

```sql
-- extraction_tasks 表
CREATE TABLE extraction_tasks (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    task_type TEXT NOT NULL,  -- 'single', 'batch', 'scheduled'
    config TEXT NOT NULL,      -- JSON
    schedule_cron TEXT,
    status TEXT NOT NULL,
    total_targets INTEGER DEFAULT 0,
    completed_targets INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- task_targets 表
CREATE TABLE task_targets (
    id TEXT PRIMARY KEY,
    task_id TEXT NOT NULL REFERENCES extraction_tasks(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    screenshot_path TEXT,
    markers TEXT NOT NULL,     -- JSON
    status TEXT NOT NULL,
    result_id TEXT,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- extraction_results 表
CREATE TABLE extraction_results (
    id TEXT PRIMARY KEY,
    target_id TEXT NOT NULL REFERENCES task_targets(id),
    source_url TEXT NOT NULL,
    data TEXT NOT NULL,         -- JSON
    confidence REAL,
    created_at INTEGER NOT NULL
);

-- marker_templates 表
CREATE TABLE marker_templates (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT,
    url_pattern TEXT,
    markers TEXT NOT NULL,     -- JSON
    is_public INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL
);

-- settings 表 (应用设置)
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at INTEGER NOT NULL
);

-- 索引
CREATE INDEX idx_tasks_status ON extraction_tasks(status);
CREATE INDEX idx_targets_task_id ON task_targets(task_id);
CREATE INDEX idx_results_target_id ON extraction_results(target_id);
```

---

## 6. 跨平台打包

### 6.1 平台特定配置

```json
// tauri.conf.json
{
  "bundle": {
    "active": true,
    "targets": ["all"],
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "identifier": "com.crawlerx.app",
    "publisher": "CrawlerX Team",
    "category": "Productivity",
    "shortDescription": "Visual web data extraction tool",
    "longDescription": "CrawlerX is a powerful visual web data extraction tool that allows you to mark and extract data from web pages using an intuitive screenshot-based interface."
  }
}
```

### 6.2 打包命令

```bash
# 开发模式
npm run tauri dev

# 构建所有平台
npm run tauri build

# 构建 macOS
npm run tauri build -- --target universal-apple-darwin

# 构建 Windows
npm run tauri build -- --target x86_64-pc-windows-msvc

# 构建 Linux
npm run tauri build -- --target x86_64-unknown-linux-gnu
```

---

## 7. 技术栈总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    CrawlerX 技术栈 (Tauri 2.0)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  前端技术栈:                                                    │
│  ├─ Vue 3.4 + TypeScript + Vite                                 │
│  ├─ Element Plus / Naive UI (UI 组件库)                         │
│  ├─ Pinia (状态管理) + VueUse (Composables)                     │
│  ├─ Vue Router 4 (路由管理)                                     │
│  └─ Tauri API (IPC 通信)                                        │
│                                                                 │
│  后端技术栈:                                                    │
│  ├─ Rust + Tauri 2.0 Core                                       │
│  ├─ Tokio (异步运行时)                                          │
│  ├─ SQLx (数据库) + SQLite (本地存储)                            │
│  ├─ fantoccini/thirtyfour (浏览器自动化)                         │
│  ├─ image + opencv (图像处理)                                    │
│  └─ tokio-cron-scheduler (任务调度)                              │
│                                                                 │
│  数据存储:                                                      │
│  ├─ SQLite (本地主存储)                                         │
│  └─ PostgreSQL (可选云端同步)                                    │
│                                                                 │
│  跨平台:                                                        │
│  └─ Windows / macOS / Linux                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 参考文档

| 文档 | 链接 |
|------|------|
| 技术栈设计 | `./06-Tauri_Tech_Stack.md` |
| 数据库设计 | `./03-DDD_Database_Design_Document.md` (SQLite 版) |
| 任务规划 | `./04-TPD_Task_Planning_Document.md` |

---

**文档变更记录**

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|----------|------|
| v1.0.0 | 2025-12-25 | 初始版本 - Tauri 2.0 CS 架构 | AI System Architect |
| v1.1.0 | 2025-12-25 | 更新为 Vue 3.0 前端技术栈 | AI System Architect |

---

*本文档定义了 CrawlerX 基于 Tauri 2.0 的完整系统架构，采用 Rust 后端 + Vue 3 前端的桌面应用架构。*
