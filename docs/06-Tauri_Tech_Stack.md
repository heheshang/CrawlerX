# CrawlerX 技术栈设计 (Tauri 2.0 + Vue 3.0)
## Technical Stack Design for Tauri 2.0

> **版本**: v1.1.0
> **日期**: 2025-12-25
> **项目代号**: CrawlerX
> **架构类型**: CS (Client-Server) Desktop Application

---

## 📋 文档信息

| 项目 | 内容 |
|------|------|
| **文档名称** | CrawlerX 技术栈设计文档 (Tauri 2.0) |
| **文档版本** | v1.0.0 |
| **创建日期** | 2025-12-25 |
| **架构类型** | Tauri 2.0 CS 桌面应用 |

---

## 1. 技术栈概述

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                   CrawlerX - Tauri 2.0 架构                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   前端层 (WebView)                       │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │  React 18    │  │  TypeScript  │  │  TailwindCSS │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │  Zustand     │  │  React Query │  │  Tauri API   │  │   │
│  │  │  (状态管理)   │  │  (数据获取)   │  │  (IPC通信)    │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↕ IPC                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   后端层 (Rust Core)                     │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │  Tauri 2.0   │  │   Tokio      │  │   Serde      │  │   │
│  │  │  (App Core)  │  │  (Runtime)   │  │  (序列化)     │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │  Vision Mod  │  │  Data Mod    │  │  Task Mod    │  │   │
│  │  │  (视觉识别)   │  │  (数据提取)   │  │  (任务调度)   │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                │
│                              ▼                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      数据存储层                           │   │
│  │                                                         │   │
│  │  ┌──────────────┐         ┌──────────────┐             │   │
│  │  │   SQLite     │   ◀──▶  │  PostgreSQL   │ (可选云端) │   │
│  │  │  (本地存储)   │   同步   │  (云端备份)   │             │   │
│  │  └──────────────┘         └──────────────┘             │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 技术栈选型对比

| 层级 | Tauri 2.0 方案 | 传统 Web 方案 | 优势 |
|------|----------------|---------------|------|
| **前端** | React + WebView | React + Browser | 相同技术栈 |
| **后端** | Rust (Native) | Node.js/Python | 性能提升 5-10x |
| **打包** | Tauri (Rust + WebView) | Electron (Chromium) | 体积减少 70% |
| **存储** | SQLite (本地) | PostgreSQL (远程) | 离线可用 |
| **集成** | 原生 OS API | Web API | 更好系统集成 |

---

## 2. 前端技术栈

### 2.1 核心框架

| 技术 | 版本 | 用途 |
|------|------|------|
| **Vue** | 3.4+ | UI 框架 |
| **TypeScript** | 5.x | 类型安全 |
| **Vite** | 5.x | 构建工具 |
| **Tauri API** | 2.0 | 前后端通信 |

### 2.2 UI 组件库

```typescript
// 推荐: Element Plus 或 Naive UI
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'

// 或使用 Naive UI (更轻量)
import NaiveUi from 'naive-ui'

// 图标库
import * as ElementPlusIconsVue from '@element-plus/icons-vue'
```

### 2.3 状态管理

```typescript
// 使用 Pinia (Vue 3 官方状态管理)
import { defineStore } from 'pinia'
import { invoke } from '@tauri-apps/api/core'

interface TaskState {
  tasks: Task[]
  loading: boolean
}

export const useTaskStore = defineStore('task', {
  state: (): TaskState => ({
    tasks: [],
    loading: false
  }),

  getters: {
    pendingTasks: (state) => state.tasks.filter(t => t.status === 'pending'),
    completedTasks: (state) => state.tasks.filter(t => t.status === 'completed')
  },

  actions: {
    async fetchTasks() {
      this.loading = true
      try {
        this.tasks = await invoke<Task[]>('get_tasks')
      } finally {
        this.loading = false
      }
    },

    async addTask(task: Task) {
      await invoke('add_task', { task })
      this.tasks.push(task)
    }
  }
})
```

### 2.4 数据获取

```typescript
// 使用 VueUse @tauri-apps/plugin-sql
import { useQuery } from '@tanstack/vue-query'
import { invoke } from '@tauri-apps/api/core'

export function useTasks() {
  return useQuery({
    queryKey: ['tasks'],
    queryFn: () => invoke<Task[]>('get_tasks'),
  })
}

// 或使用 VueUse composables
import { useAsyncState, useFetch } from '@vueuse/core'

export function useTasks() {
  return useAsyncState<Task[]>([], () => invoke('get_tasks'))
}
```

### 2.5 IPC 通信

```vue
<!-- Vue 组件中调用 Tauri Commands -->
<script setup lang="ts">
import { ref } from 'vue'
import { invoke } from '@tauri-apps/api/core'
import { listen } from '@tauri-apps/api/event'

const tasks = ref<Task[]>([])

// 调用 Rust Command
async function extractData(url: string, markers: Marker[]) {
  const result = await Invoke<ExtractResult>('extract_data', {
    url,
    screenshot: screenshotData.value,
    markers
  })
  return result
}

// 监听 Rust Event
onMounted(async () => {
  const unlisten = await listen<TaskProgress>('task-progress', (event) => {
    console.log('Task progress:', event.payload)
  })

  onUnmounted(() => {
    unlisten()
  })
})
</script>
```

### 2.6 路由管理

```typescript
// 使用 Vue Router
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  { path: '/', component: () => import('@/views/Dashboard.vue') },
  { path: '/tasks', component: () => import('@/views/Tasks.vue') },
  { path: '/tasks/:id', component: () => import('@/views/TaskDetail.vue') },
  { path: '/results', component: () => import('@/views/Results.vue') },
  { path: '/settings', component: () => import('@/views/Settings.vue') }
]

export const router = createRouter({
  history: createWebHistory(),
  routes
})
```

---

## 3. 后端技术栈 (Rust)

### 3.1 核心 Crate

```toml
[dependencies]
# Tauri 核心依赖
tauri = { version = "2.0", features = ["shell-open"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }

# 数据库
sqlx = { version = "0.7", features = ["runtime-tokio", "sqlite", "uuid", "chrono"] }
rusqlite = "0.30"

# 图像处理
image = "0.24"
opencv = { version = "0.88", features = ["opencv-4"] }

# HTTP 客户端
reqwest = { version = "0.11", features = ["json", "cookies"] }
hyper = "0.14"

# 浏览器自动化
thirtyfour = "0.31"  # Selenium for Rust
 fantoccini = "0.19"  # WebDriver client

# 异步任务
tokio-cron-scheduler = "0.9"

# 日志
tracing = "0.1"
tracing-subscriber = "0.3"

# 错误处理
anyhow = "1"
thiserror = "1"
```

### 3.2 Tauri Command 定义

```rust
// src-tauri/src/commands/vision.rs
use serde::{Deserialize, Serialize};
use tauri::State;

#[derive(Debug, Serialize, Deserialize)]
pub struct Marker {
    pub id: String,
    pub x: u32,
    pub y: u32,
    pub width: u32,
    pub height: u32,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct AnalyzeRequest {
    pub url: String,
    pub screenshot: String,  // base64
    pub markers: Vec<Marker>,
}

#[tauri::command]
pub async fn analyze_screenshot(
    req: AnalyzeRequest,
    vision_service: State<'_, VisionService>,
) -> Result<Vec<DomMapping>, String> {
    vision_service
        .analyze(&req.url, &req.screenshot, &req.markers)
        .await
        .map_err(|e| e.to_string())
}
```

### 3.3 模块组织

```
src-tauri/src/
├── main.rs                 # 入口文件
├── lib.rs                  # 库入口
├── commands/               # Tauri Commands
│   ├── mod.rs
│   ├── vision.rs           # 视觉识别命令
│   ├── data.rs             # 数据提取命令
│   ├── task.rs             # 任务管理命令
│   └── user.rs             # 用户管理命令
├── services/               # 业务服务
│   ├── mod.rs
│   ├── vision_service.rs   # 视觉识别服务
│   ├── data_service.rs     # 数据提取服务
│   ├── task_service.rs     # 任务调度服务
│   └── browser_pool.rs     # 浏览器连接池
├── models/                 # 数据模型
│   ├── mod.rs
│   ├── task.rs
│   ├── target.rs
│   └── result.rs
├── repository/             # 数据访问层
│   ├── mod.rs
│   ├── sqlite.rs           # SQLite 实现
│   └── postgres.rs         # PostgreSQL 实现 (可选)
└── utils/                  # 工具函数
    ├── mod.rs
    ├── image.rs
    └── crypto.rs
```

---

## 4. 数据存储方案

### 4.1 混合存储架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      数据存储架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  本地存储 (SQLite) - 主存储                               │   │
│  │                                                         │   │
│  │  • 用户数据      • 任务配置      • 提取模板              │   │
│  │  • 本地任务      • 缓存数据      • 临时文件              │   │
│  │  • 会话数据      • 离线队列      • 本地日志              │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                    (可选同步)                                   │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  云端存储 (PostgreSQL) - 备份/同步                       │   │
│  │                                                         │   │
│  │  • 跨设备数据同步  • 云端备份      • 团队协作            │   │
│  │  • 远程任务执行    • 历史归档      • 数据分析            │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 SQLite 本地数据库设计

```rust
// src-tauri/src/repository/sqlite.rs
use sqlx::{SqlitePool, Row};
use anyhow::Result;

pub struct SqliteRepository {
    pool: SqlitePool,
}

impl SqliteRepository {
    pub async fn new(database_url: &str) -> Result<Self> {
        let pool = SqlitePool::connect(database_url).await?;

        // 初始化表结构
        sqlx::query(
            r#"
            CREATE TABLE IF NOT EXISTS extraction_tasks (
                id TEXT PRIMARY KEY,
                name TEXT NOT NULL,
                config TEXT NOT NULL,
                status TEXT NOT NULL,
                created_at INTEGER NOT NULL,
                updated_at INTEGER NOT NULL
            );

            CREATE TABLE IF NOT EXISTS task_targets (
                id TEXT PRIMARY KEY,
                task_id TEXT NOT NULL REFERENCES extraction_tasks(id),
                url TEXT NOT NULL,
                markers TEXT NOT NULL,
                status TEXT NOT NULL,
                created_at INTEGER NOT NULL
            );
            "#
        )
        .execute(&pool)
        .await?;

        Ok(Self { pool })
    }
}
```

### 4.3 SQLite 优势

| 特性 | 说明 |
|------|------|
| **零配置** | 嵌入式数据库，无需独立服务器 |
| **轻量级** | 单文件存储，体积小 |
| **高性能** | 读写速度快，适合本地应用 |
| **跨平台** | 支持 Windows/macOS/Linux |
| **事务支持** | ACID 特性保证数据一致性 |
| **离线可用** | 完全本地运行，无网络依赖 |

---

## 5. 浏览器自动化方案

### 5.1 Rust 浏览器驱动

```toml
[dependencies]
# 方案1: fantoccini (纯 Rust WebDriver 客户端)
fantoccini = "0.19"

# 方案2: thirtyfour ( Selenium Rust 绑定)
thirtyfour = "0.31"

# 方案3: 调用外部浏览器 (headless Chrome)
# 使用 Tauri Shell Command 调用 puppeteer-cli
```

### 5.2 浏览器池实现

```rust
// src-tauri/src/services/browser_pool.rs
use fantoccini::{Client, ClientBuilder};
use std::sync::Arc;
use tokio::sync::Semaphore;

pub struct BrowserPool {
    semaphore: Arc<Semaphore>,
    max_browsers: usize,
}

impl BrowserPool {
    pub fn new(max_browsers: usize) -> Self {
        Self {
            semaphore: Arc::new(Semaphore::new(max_browsers)),
            max_browsers,
        }
    }

    pub async fn acquire(&self) -> Result<Client, fantoccini::error::CmdError> {
        let _permit = self.semaphore.acquire().await.unwrap();

        let client = ClientBuilder::native()
            .connect("http://localhost:9515")
            .await?;

        Ok(client)
    }
}
```

---

## 6. 视觉识别方案

### 6.1 Rust 图像处理库

```toml
[dependencies]
# 图像处理
image = "0.24"
imageproc = "0.23"

# OpenCV Rust 绑定
opencv = { version = "0.88", features = ["opencv-4", "contrib"] }

# 机器学习 (可选)
candle-core = "0.4"
candle-nn = "0.4"
```

### 6.2 坐标映射实现

```rust
// src-tauri/src/services/vision_service.rs
use image::{DynamicImage, ImageBuffer};
use opencv::core::{Mat, Point, Rect};

pub struct VisionService {
    browser_pool: Arc<BrowserPool>,
}

impl VisionService {
    pub async fn analyze(
        &self,
        url: &str,
        screenshot_base64: &str,
        markers: &[Marker],
    ) -> Result<Vec<DomMapping>, Box<dyn std::error::Error>> {
        // 1. 解码截图
        let screenshot = self.decode_base64_image(screenshot_base64)?;

        // 2. 获取页面 DOM
        let client = self.browser_pool.acquire().await?;
        client.goto(url).await?;
        let dom = client.source().await?;

        // 3. 坐标映射
        let mut mappings = Vec::new();
        for marker in markers {
            let dom_element = self.map_coordinate_to_dom(
                &screenshot,
                &dom,
                marker.x,
                marker.y,
                marker.width,
                marker.height,
            ).await?;

            mappings.push(dom_element);
        }

        Ok(mappings)
    }

    fn decode_base64_image(&self, base64: &str) -> Result<DynamicImage, Box<dyn std::error::Error>> {
        // Base64 解码 → 图像加载
        let bytes = base64::decode(base64)?;
        let img = image::load_from_memory(&bytes)?;
        Ok(img)
    }
}
```

---

## 7. 任务调度方案

### 7.1 本地任务调度

```toml
[dependencies]
tokio-cron-scheduler = "0.9"
```

```rust
// src-tauri/src/services/task_scheduler.rs
use tokio_cron_scheduler::{Job, JobScheduler};
use std::sync::Arc;
use tokio::sync::Mutex;

pub struct TaskScheduler {
    scheduler: Arc<Mutex<JobScheduler>>,
}

impl TaskScheduler {
    pub async fn new() -> Result<Self, Box<dyn std::error::Error>> {
        let scheduler = JobScheduler::new().await?;
        Ok(Self {
            scheduler: Arc::new(Mutex::new(scheduler)),
        })
    }

    pub async fn add_scheduled_task(
        &self,
        task_id: String,
        cron_expression: &str,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let job = Job::new_async(cron_expression, move |_uuid, _l| {
            Box::pin(async move {
                // 执行任务逻辑
                execute_task(&task_id).await;
            })
        })?;

        self.scheduler.lock().await.add(job).await?;
        Ok(())
    }
}
```

### 7.2 异步任务处理

```rust
// 使用 Tokio 运行时处理异步任务
#[tauri::command]
pub async fn create_task(task: Task) -> Result<String, String> {
    // 保存到 SQLite
    let task_id = repository::save_task(&task).await?;

    // 异步执行任务
    tokio::spawn(async move {
        if let Err(e) = execute_task_inner(&task_id).await {
            eprintln!("Task execution error: {}", e);
        }
    });

    Ok(task_id)
}

async fn execute_task_inner(task_id: &str) -> Result<(), Box<dyn std::error::Error>> {
    // 更新任务状态
    repository::update_task_status(task_id, "running").await?;

    // 执行数据提取
    let result = data_service::extract_data(task_id).await?;

    // 保存结果
    repository::save_result(task_id, &result).await?;

    // 更新任务状态
    repository::update_task_status(task_id, "completed").await?;

    Ok(())
}
```

---

## 8. 跨平台打包

### 8.1 Tauri 配置

```json
// tauri.conf.json
{
  "build": {
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build",
    "devUrl": "http://localhost:1420",
    "frontendDist": "../dist"
  },
  "bundle": {
    "active": true,
    "targets": ["all"],
    "icon": ["icons/32x32.png", "icons/128x128.png", "icons/128x128@2x.png", "icons/icon.icns", "icons/icon.ico"]
  },
  "app": {
    "windows": [
      {
        "title": "CrawlerX",
        "width": 1400,
        "height": 900,
        "resizable": true,
        "fullscreen": false,
        "minWidth": 1000,
        "minHeight": 600
      }
    ],
    "security": {
      "csp": null
    }
  }
}
```

### 8.2 打包命令

```bash
# 开发模式
npm run tauri dev

# 构建生产版本
npm run tauri build

# 打包特定平台
npm run tauri build -- --target universal-apple-darwin    # macOS
npm run tauri build -- --target x86_64-pc-windows-msvc     # Windows
npm run tauri build -- --target x86_64-unknown-linux-gnu   # Linux
```

---

## 9. 技术栈优势

### 9.1 性能对比

| 指标 | Electron (原方案) | Tauri 2.0 (新方案) | 提升 |
|------|-------------------|-------------------|------|
| **安装包大小** | ~150 MB | ~10 MB | **93% ↓** |
| **内存占用** | ~200 MB | ~50 MB | **75% ↓** |
| **启动时间** | ~3s | ~0.5s | **83% ↓** |
| **CPU 使用率** | 基准 | **-40%** | 更低 |

### 9.2 开发体验

| 方面 | Tauri 2.0 优势 |
|------|----------------|
| **前端开发** | 完全使用熟悉的 React/TypeScript |
| **类型安全** | Rust + TypeScript 双重类型保护 |
| **热重载** | 开发模式支持前端热更新 |
| **调试** | 支持 Chrome DevTools + Rust 调试器 |
| **分发** | 单文件可执行，无需用户安装依赖 |

---

## 10. 技术栈总览

```
┌─────────────────────────────────────────────────────────────────┐
│                    CrawlerX 技术栈总览                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  前端: Vue 3.4 + TypeScript + Vite + Element Plus                │
│  后端: Rust + Tauri 2.0 + Tokio                                  │
│  本地存储: SQLite + SQLx                                         │
│  云端同步: PostgreSQL (可选)                                     │
│  图像处理: image + opencv                                        │
│  浏览器自动化: fantoccini / thirtyfour                           │
│  任务调度: tokio-cron-scheduler                                  │
│  状态管理: Pinia + VueUse / @tanstack/vue-query                 │
│  路由管理: Vue Router 4                                          │
│  IPC通信: Tauri Commands & Events                               │
│  跨平台: Windows / macOS / Linux                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. 参考文档

| 文档 | 链接 |
|------|------|
| Tauri 官方文档 | https://tauri.app/ |
| Tauri 2.0 迁移指南 | https://v2.tauri.app/start/migrate |
| SQLx 文档 | https://docs.rs/sqlx/ |
| fantoccini 文档 | https://docs.rs/fantoccini/ |

---

**文档变更记录**

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|----------|------|
| v1.0.0 | 2025-12-25 | 初始版本 - Tauri 2.0 + React 技术栈 | AI System Architect |
| v1.1.0 | 2025-12-25 | 更新为 Vue 3.0 技术栈 | AI System Architect |

---

*本文档定义了 CrawlerX 基于 Tauri 2.0 的完整技术栈，采用 Rust 后端 + React 前端的桌面应用架构。*
