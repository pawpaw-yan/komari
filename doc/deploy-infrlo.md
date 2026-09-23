# Komari 部署到 infrlo 容器

infrlo 容器只提供两个配置项：**Build Command** 和 **Run Command**（默认值分别为
`pip install -r requirements.txt` 和 `python app.py`）。

Komari 是 Go 预编译单二进制程序，**前端已内嵌**，运行时只依赖一个可写的工作目录，
因此采用「Python 引导脚本」方案：`app.py` 在构建期/运行期从 GitHub Releases 下载
对应架构的二进制，然后用 `execv` 把 `python app.py` 进程替换为 `./komari server`，
平台两条命令基本保持默认即可。

## 配置

| 配置项 | 填写内容 |
|---|---|
| Build Command | `pip install -r requirements.txt && python app.py --download-only` |
| Run Command | `python app.py` |
| 端口 | komari 默认监听 `0.0.0.0:25774`；若平台注入 `PORT` 环境变量则自动跟随 |

> 即使 Build Command 保持默认 `pip install -r requirements.txt` 也能部署成功
> （该文件为空依赖，仅作兼容占位），二进制会推迟到运行期首次启动时下载；
> 但建议按上表在构建期预下载，启动更快，也不依赖运行环境的外网连通性。

## 可选环境变量（在 infrlo 容器设置页添加）

| 变量 | 说明 | 默认值 |
|---|---|---|
| `PORT` | 监听端口（平台通常会自动注入） | `25774` |
| `KOMARI_VERSION` | 指定版本号，如 `1.5.0-fix1` | 自动取最新 release |
| `KOMARI_DOWNLOAD_URL` | 直接指定二进制下载地址，可填 ghproxy 等镜像加速 | GitHub 官方地址 |
| `KOMARI_LISTEN` | 监听地址，如 `0.0.0.0:8080` | `0.0.0.0:$PORT` |
| `KOMARI_DATABASE` | SQLite 文件路径 | `./data/komari.db` |
| `KOMARI_ARGS` | 附加给 `komari server` 的参数（空格分隔） | 空 |

## 数据持久化

数据库默认写在容器工作目录 `./data/komari.db`。infrlo 这类容器重启/重新部署后
**文件系统通常是临时的**，面板数据（账号、探针列表、记录）会丢失：

- 若平台提供持久化存储/挂载目录，设置 `KOMARI_DATABASE=/持久化路径/komari.db`；
- 否则每次重新部署需重新执行首次安装向导（浏览器打开面板首页即可进入安装页）；
- 可用面板自带的备份/恢复功能在两次部署间迁移数据。

## 其他注意事项

- **WebSocket**：探针（agent）通过 WebSocket 连接面板，部署后需确认平台已开启
  WebSocket 支持，否则探针连不上。
- **架构自动适配**：`app.py` 会根据容器架构自动选择 `komari-linux-{amd64,arm64,386,loong64,riscv64}`。
- **下载失败**：若容器访问 GitHub 受限，设置 `KOMARI_DOWNLOAD_URL` 走镜像，
  例如 `https://ghproxy.net/https://github.com/komari-monitor/komari/releases/download/1.5.0-fix1/komari-linux-amd64`。
- **探针端**：部署好面板后，在面板后台复制 `install-komari.sh` 命令安装各节点探针，
  连接地址填 infrlo 分配的域名，WebSocket 端口与面板同端口。

## 文件说明

| 文件 | 作用 |
|---|---|
| `app.py` | Python 引导脚本：下载二进制 → exec 启动 komari |
| `requirements.txt` | 空依赖占位，兼容平台默认构建命令 |
| `Dockerfile` | 原有 Docker 部署方式，与 infrlo 方案互不影响 |
