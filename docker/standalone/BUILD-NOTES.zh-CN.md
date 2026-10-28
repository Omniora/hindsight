# main 分支本地镜像构建记录

日期：2026-10-21。环境：macOS、Docker Desktop、Linux ARM64 构建目标。

## 最终产物

- 镜像标签：`hindsight-local:main`。
- 镜像 ID：`sha256:074eb1912607c13270dc7e4ffc3da25799bac2dfdde36d0de8bd74cabcc0e817`。
- 架构：`linux/arm64`。
- 大小：2,483,724,904 字节，约 2.48 GB。
- 源码 HEAD：`682bfc80ad55`，另包含下述本地 Dockerfile 补丁。
- API 包版本：`hindsight-api-slim==1.0.0`。
- 前端构建版本：Next.js 16.4.2。
- 本次完整构建成功，无新增失败；最终模型预下载阶段输出 `Models cached successfully`。
- 完整构建日志保存在本机 `/tmp/hindsight-build-current.log`，该路径为临时文件。

## 构建入口

在仓库根目录执行，使用本机已验证可用的 HTTP 代理端口；其他机器需要替换代理地址。

```bash
HTTP_PROXY=http://127.0.0.1:12334 \
HTTPS_PROXY=http://127.0.0.1:12334 \
docker build \
  --progress=plain \
  -f docker/standalone/Dockerfile \
  --target standalone \
  -t hindsight-local:main \
  .
```

`standalone` 包含 API（8888）和管理界面（9999）。本次保留 Dockerfile 默认的本地 ML 依赖和模型预下载配置。
镜像内容来自本地工作区，包含本次 Dockerfile 修改，并非仅由 main 的提交号决定。

## 问题与处理

### 1. 基础镜像认证令牌获取超时

现象：拉取 `python:3.11-slim`、`node:24-slim` 的元数据时，访问 `auth.docker.io/token` 超时。

排查结果：IPv4、IPv6 直连均超时；显式经过 `http://127.0.0.1:12334` 的认证请求返回 HTTP 200。
设置 Docker Desktop 代理后，日志显示镜像仓库请求已经走代理，但构建客户端的令牌请求仍然直连。

处理：保留 Docker Desktop 的代理设置，同时在调用 `docker build` 时设置 `HTTP_PROXY` 和 `HTTPS_PROXY`。
对比测试中，添加这两个环境变量后，两个基础镜像均下载成功。
这两个变量用于宿主机侧的 Docker 命令；不要把宿主机的 `127.0.0.1` 代理地址直接当作构建容器内的代理地址。

### 2. APT 下载失败，并出现仓库签名报错

现象：`apt-get update` 下载 `trixie-updates` 和 `trixie-security` 的 `InRelease` 文件时，代理返回 HTTP 500，错误包含 `connection reset by peer`；随后出现 `is no longer signed`，退出码为 100。

排查结果：在同一个 Python 基础镜像中，原来的 HTTP 软件源复现失败；切换为 HTTPS 后，三个仓库的索引全部下载成功。
这里的签名报错是签名元数据未能下载的后果。

处理：修改 `docker/standalone/Dockerfile` 的 `api-builder`、`api-only` 和 `standalone` 三个阶段：

- 将 `/etc/apt/sources.list.d/debian.sources` 中的 `http://deb.debian.org` 替换为 `https://deb.debian.org`。
- 为 `apt-get update` 添加 `-o Acquire::Retries=3`。
- 保留原有签名验证和软件包升级步骤。

验证：最小 Docker 构建下载索引成功；完整构建中的系统依赖安装成功；`docker build --check` 无警告。

### 3. Next.js 字体构建报 Module not found

现象：`next/font/google` 生成的 Inter 字体样式报 `Can't resolve '@vercel/turbopack-next/internal/font/google/font'`，同时出现 `next/font/google queries have exactly one entry`。

分析：错误特征与 Next.js 上游的字体 URL 解析问题吻合。Google Fonts 偶尔返回带查询参数的无扩展名 URL，可能触发该错误。
详见 [Next.js issue #99114](https://github.com/vercel/next.js/issues/99114)。用户日志未包含完整字体响应，因此本次无法完全确认触发条件。

处理与验证：保留原有前端依赖和源码，重新执行完整构建。本次实际使用 Next.js 16.4.2，Turbopack 编译、TypeScript 检查及 58 个静态页面生成全部通过。
未安装报错中提到的内部虚拟模块，也未切换打包器。此结果说明本次重跑已通过，不能保证未来每次远程字体请求都成功。

## 在现有 Compose 中使用本地镜像

将 `hindsight` 服务的镜像配置替换为：

```yaml
image: hindsight-local:main
pull_policy: never
```

数据库和扩展已初始化时，可以只重建应用容器：

```bash
docker compose \
  -f docker/docker-compose/pgroonga/docker-compose.yaml \
  up -d --no-deps --force-recreate hindsight
```

本记录中的 ARM64 镜像适用于对应架构。目标服务器若为 AMD64，需要重新按 `--platform linux/amd64` 构建。

## 验证范围

- 完整 `standalone` 镜像构建和导出成功。
- Python 安装环境的 `uv pip check` 检查 210 个包，全部兼容。
- `hindsight_api.main` 在最终镜像中可导入。
- 使用 `--network none`、`HF_HUB_OFFLINE=1`、`TRANSFORMERS_OFFLINE=1` 验证两个模型：
  - `BAAI/bge-small-en-v1.5` 加载成功，实际编码得到 `(1, 384)` 向量，所有值有限。
  - `cross-encoder/ms-marco-MiniLM-L-6-v2` 加载成功，实际计算一组文本对的分数，结果有限。
- 启动临时容器，仅启用管理界面；`/dashboard`、`/login`、`/api/health` 均返回 HTTP 200。
  `/en/dashboard` 返回到 `/dashboard` 的 307 是当前无语言前缀路由的正常行为。
- 管理界面测试关闭了 API，因此上述检查不代表 PostgreSQL、pgroonga、真实 LLM 请求已通过端到端验证。
- 本次未替换现有部署，也未修改 Compose 文件；临时测试容器已停止并自动删除。
- `git diff --check` 通过，按项目 code-review 清单复核 Dockerfile 修改，未发现必须修复的问题。

项目 lint 脚本此前因宿主机缺少 `uv` 未能执行；Docker 构建使用镜像内部安装的 `uv`，不依赖宿主机安装。
