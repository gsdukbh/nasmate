# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

NAStool 是一个 NAS 媒体库自动化管理工具，基于 Python 3.10 + Flask 构建。核心功能包括：PT 站点订阅与刷流、媒体文件自动识别与整理、下载器管理、媒体服务器库刷新、多渠道消息推送。

## 运行与开发

### 本地启动

```bash
pip install -r requirements.txt
export NASTOOL_CONFIG=/path/to/config/config.yaml
python run.py
```

Flask 默认监听 `0.0.0.0:3000`，可在 `config.yaml` 的 `app.web_port` 修改。

### Docker 启动

```bash
# 构建镜像
docker build -t nas-tools .

# 使用 compose 启动（挂载 config 目录和媒体目录）
docker compose -f docker/compose.yml up -d
```

### 运行测试

```bash
# 运行所有测试
python -m pytest

# 运行单个测试套件
python -m tests.run

# 运行特定测试
python -m pytest tests/test_metainfo.py
```

测试目前主要覆盖 `app/media/meta/` 的媒体名称识别逻辑（`tests/cases/meta_cases.py`）。

## 配置系统

- **主配置**：`NASTOOL_CONFIG` 环境变量指向的 YAML 文件（默认 `/config/config.yaml`）
- **Config 单例**：`config.py` 中的 `Config` 类，通过 `Config().get_config('section')` 读取各节
- **全局常量**：`config.py` 顶部定义了所有时间间隔常量（`PT_TRANSFER_INTERVAL`、`RSS_CHECK_INTERVAL` 等）
- **配置热重载**：`run.py` 用 watchdog 监听 `config.yaml` 变化，自动调用 `Config().init_config()` 并重启调度器

关键环境变量：`NASTOOL_CONFIG`、`NASTOOL_LOG`、`NASTOOL_AUTO_UPDATE`、`TZ`

## 代码架构

### 启动流程（`run.py`）

```
init_system() → init_db / update_db / init_data / check_config
start_service() → DisplayHelper / run_scheduler / run_monitor / BrushTask /
                  RssChecker / TorrentRemover / SpeedLimiter / IndexerHelper / ChromeHelper
App.run() → Flask Web 服务器
```

### 核心模块职责

| 模块 | 路径 | 职责 |
|------|------|------|
| 媒体识别 | `app/media/meta/` | 从文件名/标题解析媒体元数据（年份、季集、分辨率等） |
| 元数据获取 | `app/media/media.py` | 调用 TMDB/豆瓣/Fanart API |
| 文件转移 | `app/filetransfer.py` | 硬链接/软链接/复制/移动文件，重命名为规范格式 |
| 目录监控 | `app/sync.py` | watchdog 监控同步目录，触发文件转移 |
| 下载管理 | `app/downloader/` | 统一下载器接口，下层支持 qBittorrent/Transmission/Aria2/115/PikPak |
| 索引器 | `app/indexer/` | 站点搜索聚合，支持 Jackett/Prowlarr/内置爬虫 |
| 媒体服务器 | `app/mediaserver/` | 触发 Emby/Jellyfin/Plex 刷库，处理 Webhook 事件 |
| 消息推送 | `app/message/` | 向 Telegram/WeChat/Bark/Slack 等渠道推送通知 |
| 定时调度 | `app/scheduler.py` | APScheduler 管理所有周期任务 |
| 刷流 | `app/brushtask.py` | PT 站点刷流养种逻辑 |
| RSS 订阅 | `app/rss.py` + `app/rsschecker.py` | 媒体订阅与 RSS 检查 |

### Web 层（`web/`）

- `web/main.py`：Flask App 实例，注册所有路由和蓝图
- `web/action.py`：页面交互路由（表单提交、UI 操作）
- `web/apiv1.py`：JSON REST API 蓝图（`/api/v1/`）
- `web/backend/`：业务逻辑实现（用户认证、工具函数等）
- `web/templates/`：Jinja2 模板
- `web/static/`：前端静态资源

### 抽象层模式

下载器、索引器、消息渠道、媒体服务器均采用同一模式：
- `client/_base.py` 定义抽象接口
- `client/xxx.py` 实现具体客户端
- `xxx.py`（同级）作为门面类，根据配置实例化对应客户端

### 单例与实例管理

业务服务类使用 `app/utils/commons.py` 中的 `INSTANCES` 字典统一管理单例，配置热重载时批量调用 `init_config()` 方法重新初始化。

### 数据库

SQLite + SQLAlchemy ORM，模型定义在 `app/db/models.py`，初始化/迁移逻辑在 `app/db/main_db.py`。数据库文件默认位于配置目录下。
