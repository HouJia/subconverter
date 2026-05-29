# 技术迭代：subconverter — NAS 部署与 NPM `/subapi` 暴露

> [!CAUTION]
> **本文档已迁移。** 请以 **[HouJia/SubConverter-Extended](https://github.com/HouJia/SubConverter-Extended)** 仓库 `hjsmaster` 分支下的同名文件为准：  
> `docs/技术迭代-NAS部署与NPM暴露.md`  
> 功能说明见 Extended：`docs/功能与访问入口.md`。本文件仅作归档副本。

> **本仓库职责**：subconverter 服务在 NAS 上监听、Docker 运行、API 路径约定。  
> **基线（2026-05-28）**：`hjsmaster` 基于 **SubConverter-Extended**（含 mihomo bridge、HTML `/version` 页等）。  
> **反代配置**：见 `nginx-proxy-manager` 仓库 `scripts/nas-qnap-npm-phase1-final.mjs`。  
> **前端**：见 `sub-web` 仓库（`DEFAULT_BACKEND` 指向 `/subapi/sub?`）。  
> **版本**：2026-05-28

## 目录

- [1. 服务角色](#1-服务角色)
  - [1.1 NPM `/subapi` 反代（必配）](#11-npm-subapi-反代必配)
  - [1.2 `/version` 页静态资源（子路径，非内联）](#12-version-页静态资源子路径非内联)
- [2. NAS 现状（参考）](#2-nas-现状参考)
- [3. 配置检查](#3-配置检查)
- [4. 本仓库交付物](#4-本仓库交付物)
- [5. 部署 / 升级](#5-部署--升级)
  - [5.1 方式 A：本机 Docker 构建（交叉编译）](#51-方式-a本机-docker-构建交叉编译)
  - [5.2 方式 B：NAS 上构建（无需本机 Docker）](#52-方式-bnas-上构建无需本机-docker)
- [6. 验收（subconverter 视角）](#6-验收subconverter-视角)
- [7. 外部配置与 max_allowed_rulesets](#7-外部配置与-max_allowed_rulesets)
- [8. 常见现象](#8-常见现象)
- [9. 跨项目依赖](#9-跨项目依赖)
- [10. 变更记录](#10-变更记录)

## 1. 服务角色

subconverter 提供订阅转换 HTTP API，默认端口 **25500**。

| 端点 | 用途 |
|------|------|
| `GET /version` | Extended 版本信息页（HTML，浏览器访问） |
| `GET /version/favicon-light.svg` | 版本页 logo / favicon（**独立 HTTP 资源**） |
| `GET /version/favicon-dark.svg` | 深色主题 favicon |
| `GET /version.txt` | 纯文本版本（**sub-web 页眉**、脚本健康检查） |
| `GET /sub?target=...&url=...` | 转换订阅 |
| `GET /dashboard` | Extended 统计面板（需在 pref 中启用 statistics） |

外网不直接暴露 `25500`，由 NPM 映射为：

- `https://<域名>:<HTTPS端口>/subapi/version` → 容器 `/version`
- `https://<域名>:<HTTPS端口>/subapi/version/favicon-light.svg` → 容器 `/version/favicon-light.svg`
- `https://<域名>:<HTTPS端口>/subapi/version.txt` → 容器 `/version.txt`
- `https://<域名>:<HTTPS端口>/subapi/sub?...` → 容器 `/sub?...`

### 1.1 NPM `/subapi` 反代（必配）

在 **Nginx Proxy Manager** 中为 subconverter 增加 **Custom Location**（或等价 Advanced 配置），**必须**用前缀剥离方式覆盖 **整个** `/subapi/`，不能只配单条 `/subapi/version`。

**Custom Location 示例**（Location = `/subapi/`，Forward = `http://<NAS内网IP>:25500/`）：

```nginx
# NPM → Proxy Host → Custom Locations → /subapi/
location ^~ /subapi/ {
    proxy_pass http://<NAS-内网IP>:25500/;   # 末尾 / 表示剥掉 /subapi 前缀
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Prefix /subapi;
}
```

| 检查项 | 正确 | 错误 |
|--------|------|------|
| `proxy_pass` 末尾 | 有 `/`（剥前缀） | 无 `/`（路径叠成 `/subapi/sub` 404） |
| Location 范围 | `^~ /subapi/` 整前缀 | 仅 `/subapi/version` 一条 |
| 静态资源 | `/subapi/version/favicon-*.svg` 可 200 | 只反代 HTML，图标 404 |

**验收命令**（将域名与端口换成你的）：

```bash
curl -sI "https://<域名>:<端口>/subapi/version/favicon-light.svg" | head -1
# 期望 HTTP/2 200 或 HTTP/1.1 200 OK

curl -s "https://<域名>:<端口>/subapi/version.txt"
# 期望一行纯文本版本
```

### 1.2 `/version` 页静态资源（子路径，非内联）

版本页 HTML 使用 **Extended 官方子路径**（与 upstream 一致）：

- 页面内：`<img src="/version/favicon-light.svg">`、`<link href="/version/favicon-dark.svg">`
- 容器内路由：`GET /version/favicon-*.svg` 返回 SVG 文件

经 NPM 访问时，浏览器请求 **`/subapi/version/favicon-light.svg`**。服务端通过以下机制补全前缀：

1. **推荐**：NPM 发送 `X-Forwarded-Prefix: /subapi`，`page_assets::rewriteLocalAssetPaths` 将 HTML 内 `/version/...` 改写为 `/subapi/version/...`
2. **兜底**：HTML 内 `<base>` 脚本按 `pathname` 推断（如 pathname 以 `/subapi/version` 结尾则 `<base href="/subapi/version/">`）

**不要**使用内联 SVG 替代上述 HTTP 资源；部署与排错以 **NPM + 子路径** 为准。

## 2. NAS 现状（参考）

| 项 | 值 |
|----|-----|
| 容器名 | `subconverter` |
| 镜像 | `subconverter:nas-amd64`（Extended 根 `Dockerfile` + NAS overlay） |
| 端口 | `0.0.0.0:25500→25500` |
| 内网访问 | `http://<NAS内网IP>:25500/` |
| 部署脚本 | 见 [§5](#5-部署--升级) |

## 3. 配置检查

### 3.1 监听地址

NAS 叠加配置见 `deploy/nas/pref.toml`（构建时 COPY 进镜像 `/base/pref.toml`）：

```toml
[server]
listen = "0.0.0.0"
port = 25500

[advanced]
max_allowed_rulesets = 256
```

### 3.2 与 sub-web 的 URL 关系

| 内容 | 性质 | 说明 |
|------|------|------|
| 路径 `/subapi`、`/subapi/sub?`、`/subapi/version.txt` | **固定约定** | NPM 与 sub-web 写死，**不能改** |
| `https://<你的域名>:<HTTPS端口>` | **部署时必填** | 公网域名 + HTTPS 端口 |

**sub-web 部署时必须配置真实后端**（见 `sub-web` 仓库 `.env`）：

```bash
VITE_SUBCONVERTER_DEFAULT_BACKEND=https://<你的域名>:<HTTPS端口>/subapi
```

页眉版本：sub-web 请求 **`/subapi/version.txt`**（纯文本）。

## 4. 本仓库交付物

| 项 | 说明 |
|----|------|
| 根 `Dockerfile` | Extended 多阶段构建（Go mihomo bridge + C++） |
| `deploy/nas/Dockerfile` | 薄 overlay：`pref.toml`、`houjia-template.ini` |
| `deploy/nas/deploy-to-qnap.sh` | **方式 A**：本机 Docker 构建 → save/load → NAS |
| `deploy/nas/deploy-build-on-qnap.sh` | **方式 B**：rsync 源码到 NAS，在 NAS 上 docker build |
| `src/handler/page_assets.h` | NPM 子路径：`X-Forwarded-Prefix` + HTML 资源路径改写 |

## 5. 部署 / 升级

### 5.1 方式 A：本机 Docker 构建（交叉编译）

**适用**：开发机为 macOS（含 Apple Silicon），需在本地交叉编译 **linux/amd64** 再导入 NAS。

**需要**：本机 **Docker Desktop 已启动**（脚本调用 `docker buildx`）。

```bash
cd subconverter   # 仓库根
./deploy/nas/deploy-to-qnap.sh
```

流程：① 本机 `docker buildx` 两阶段镜像 → ② `docker save | ssh nas docker load` → ③ SSH 重建容器。

**为何常用本机 Docker**：QNAP 上完整编译 Extended（Go + C++）耗时长、占内存；在 Mac 上 buildx 交叉编译后只传镜像更稳。

### 5.2 方式 B：NAS 上构建（无需本机 Docker）

**适用**：本机未装 / 未启动 Docker，但 NAS 上 Container Station 正常。

**需要**：本机可 `ssh nas-qnap`，NAS 上 Container Station 的 `docker build` 可用（脚本使用原生 `docker build`，非 buildx，避免 QNAP 权限问题）。

```bash
cd subconverter
./deploy/nas/deploy-build-on-qnap.sh
```

流程：① `rsync` 源码到 NAS → ② SSH 在 NAS 上两阶段 `docker build` → ③ 重建容器。

环境变量（可选）：

| 变量 | 默认 | 说明 |
|------|------|------|
| `NAS_HOST` | `nas-qnap` | SSH 主机名 |
| `REMOTE_DIR` | `/share/CACHEDEV1_DATA/Containers/subconverter-build` | NAS 上的构建目录 |
| `VERSION` | `1.1.9+houjia.2` | 写入镜像的版本字符串 |

### 5.3 升级后自检

```bash
ssh nas-qnap "curl -s http://127.0.0.1:25500/version.txt"
ssh nas-qnap "curl -sI http://127.0.0.1:25500/version/favicon-light.svg | head -1"
```

## 6. 验收（subconverter 视角）

- [ ] 内网 `25500/version.txt` 返回纯文本版本行（含构建 hash）
- [ ] 内网 `/version/favicon-light.svg` 返回 SVG
- [ ] 经 NPM `/subapi/version` 页面 logo 与 favicon 正常
- [ ] 经 NPM `/subapi/version/favicon-light.svg` 返回 200
- [ ] 经 NPM `/subapi/sub?` + 真实 `url` 可转换
- [ ] 公网不直接暴露 25500

## 7. 外部配置与 max_allowed_rulesets

上游默认 **`max_allowed_rulesets = 64`**。自建 NAS 镜像使用 **256**（`settings.h` + `deploy/nas/pref.toml`）。

## 8. 常见现象

| 现象 | 说明 |
|------|------|
| `deploy-to-qnap.sh` 报 `docker.sock` 不存在 | 本机 Docker 未启动；改用 [§5.2](#52-方式-bnas-上构建无需本机-docker) |
| `/subapi/version` 图标破损 | NPM 未配整段 `/subapi/` 或未发 `X-Forwarded-Prefix`；见 [§1.1](#11-npm-subapi-反代必配) |
| sub-web 页眉 HTML 乱码 | 误请求 `/version`；应使用 `/version.txt` |
| `No nodes were found!` | API 已通；订阅 `url` 无效或拉取失败 |

## 9. 跨项目依赖

| 项目 | 分支 |
|------|------|
| subconverter | **`hjsmaster`** |
| sub-web | **`hjsmaster`** |
| nginx-proxy-manager | `feature/nas-qnap-phase1-proxy`（或你的 NPM 配置分支） |

## 10. 变更记录

| 日期 | 说明 |
|------|------|
| 2026-05-16 | 初版：NAS + NPM `/subapi` |
| 2026-05-28 | Extended 基线、两阶段 Docker、`/version.txt` |
| 2026-05-28 | 版本页改回 **子路径 favicon** + NPM 部署指南；新增 NAS 本机构建脚本 |
