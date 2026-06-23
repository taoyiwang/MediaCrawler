# CLAUDE.md — MediaCrawler 开发指南

## 项目简介

MediaCrawler 是基于 Playwright 浏览器自动化的**多平台自媒体爬虫**，支持小红书、抖音、快手、B站、微博、贴吧、知乎的公开内容抓取。

核心技术：通过 CDP 协议连接用户本地 Chrome 浏览器，复用登录态，无需 JS 逆向。

## 环境要求

- **Python** >= 3.11，< 3.14（推荐 3.11~3.12，3.13 部分包无 wheel）
- **Node.js** >= 16（抖音、知乎签名需要）
- **Chrome** >= 144（CDP 模式）

> ⚠️ **Python 3.14 警告**：httpx 和 xhshow 在 3.14 上存在兼容问题。如果必须使用 3.14，本项目已修复 `xhshow==0.2.0` 兼容性，详见下文"已知坑点"。

## 快速开始

```shell
# 1. 安装 uv（推荐，速度快）
pip install uv

# 2. 同步依赖（自动管理 Python 版本和虚拟环境）
uv sync

# 3. 配置环境变量（仅数据库存储时需要）
copy .env.example .env

# 4. 配置 Chrome 远程调试
# Chrome 地址栏输入: chrome://inspect/#remote-debugging
# 勾选 "Allow remote debugging for this browser instance"

# 5. 运行
uv run main.py --platform xhs --lt qrcode --type search
```

## 运行模式

```shell
# 关键词搜索
uv run main.py --platform xhs --lt qrcode --type search --keywords "关键词1,关键词2"

# 指定帖子爬取
uv run main.py --platform xhs --lt qrcode --type detail --specified_id "帖子URL1,帖子URL2"

# 博主主页爬取（需要含 xsec_token 的完整 URL）
uv run main.py --platform xhs --lt qrcode --type creator --creator_id "博主主页URL"

# WebUI 可视化界面
uv run uvicorn api.main:app --port 8080 --reload

# 其他平台: dy(抖音), ks(快手), bili(B站), wb(微博), tieba(贴吧), zhihu(知乎)
```

## 关键配置 (`config/base_config.py`)

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `PLATFORM` | `"xhs"` | 目标平台 |
| `CRAWLER_TYPE` | `"search"` | 爬取模式 |
| `LOGIN_TYPE` | `"qrcode"` | 登录方式: qrcode/phone/cookie |
| `CRAWLER_MAX_NOTES_COUNT` | `15` | 最多爬取帖子数 |
| `CRAWLER_MAX_COMMENTS_COUNT_SINGLENOTES` | `10` | 每帖最多评论数 |
| `ENABLE_GET_COMMENTS` | `True` | 是否爬取评论 |
| `ENABLE_CDP_MODE` | `True` | CDP 模式（推荐） |
| `SAVE_DATA_OPTION` | `"jsonl"` | 存储格式 |
| `CRAWLER_MAX_SLEEP_SEC` | `2` | 请求间隔（秒，切勿设为0） |

平台特定配置在 `config/xhs_config.py` 等文件中（如 `XHS_CREATOR_ID_LIST`）。

## 登录方式对比

| 方式 | 适用场景 | 操作 |
|------|----------|------|
| `qrcode` | 首次使用/Chrome 未登录 | 扫码登录 |
| `cookie` | Chrome 已登录 | 复制 web_session cookie 到 `COOKIES` 配置 |
| `phone` | 手机号登录 | 需要配合 Redis 接收验证码 |

**推荐 cookie 模式**：如果 Chrome 已经登录了小红书，从 DevTools → Application → Cookies 复制 `web_session` 值，设置：
```python
LOGIN_TYPE = "cookie"
COOKIES = "web_session=你复制的值"
```

## 数据输出

- **JSONL**（默认）：`data/{platform}/jsonl/` 目录
- **DB/Postgres/SQLite**：需要先配置 `.env` 中的数据库信息，然后 `uv run main.py --init_db`

## 已知坑点 & 解决方案

### 1. xhshow 签名库兼容性（Python 3.13/3.14）
- **现象**：`'float' object has no attribute 'encode'` 或 `multiple values for argument 'sign_state'`
- **原因**：`xhshow==0.2.0` 在 `build_payload_array` 新增了 `m_value` 参数，原有 monkey-patch 参数错位
- **状态**：`media_platform/xhs/playwright_sign.py` 已修复 ✅

### 2. aiomysql 编译失败
- **现象**：`error: Microsoft Visual C++ 14.0 or greater is required`
- **原因**：`asyncmy`/`aiomysql` 需要 C++ 编译器和 MySQL 客户端库
- **解决**：如果不需要 MySQL 存储（用 JSONL/CSV），忽略即可。`var.py` 已改为可选导入 ✅

### 3. uv 下载的 Python 缺少 DLL
- **现象**：`.venv\Scripts\python.exe` 无法启动，弹窗"找不到 python3xx.dll"
- **原因**：uv 下载的嵌入式 Python 在某些 Windows 版本上缺少运行时 DLL
- **解决**：用 `python -m venv .venv` 创建环境，`pip install` 安装依赖

### 4. CDP 模式二维码登录超时
- **现象**：`Timeout 30000ms exceeded waiting for qrcode-img`
- **原因**：Chrome 已登录状态不弹登录框 / 小红书页面结构变更
- **解决**：改用 cookie 模式登录

### 5. 依赖安装策略
- 推荐用 `uv sync` 自动管理所有依赖
- 如果 `uv sync` 失败（权限/编译问题），创建标准 venv 后逐个安装
- `asyncmy` 编译失败可跳过，不影响 JSONL/CSV 存储

## 架构概览

```
main.py              # 入口，命令行解析，创建爬虫工厂
config/              # 平台配置（base + 各平台特定）
media_platform/      # 各平台爬虫实现（xhs/douyin/kuaishou/bilibili/weibo/tieba/zhihu）
  xhs/
    core.py          # 爬虫主逻辑（CDP/Playwright 启动、页面管理、爬取调度）
    client.py        # API 客户端（HTTP 请求、签名）
    login.py         # 登录流程（二维码/手机/邮箱/Cookie）
    playwright_sign.py  # xhshow 签名 + monkey-patch（API 签名核心）
    help.py          # URL 解析、签名辅助
tools/               # 通用工具（CDP 浏览器管理、文件写入、词云生成）
database/            # 数据库访问层
store/               # 各平台数据存储逻辑
cache/               # 缓存（Redis/本地）
proxy/               # 代理 IP 池
api/                 # WebUI FastAPI 后端
data/                # 默认爬取数据输出目录
```

## 调试技巧

```shell
# 验证环境是否正常
python -c "import config; print(config.PLATFORM)"

# 测试 xhshow 签名（最常见的报错点）
python -c "
from media_platform.xhs.playwright_sign import sign_with_xhshow
signs = sign_with_xhshow('/api/sns/web/v1/user/selfinfo', {}, 'a1=test;web_session=test', 'GET')
print('OK:', list(signs.keys()))
"
```
