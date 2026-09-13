# LM Studio + noVNC 容器部署手册（Ubuntu 22.04）

> 2026-09-12 整理 ｜ LM Studio 0.4.24-1 ｜ 依据 .253 生产容器实排经验
> 头号坑：**libatomic1 必装**——缺它 LM Studio 所有原生子进程 exit 127，硬件检测全灭，GUI 看不到 GPU

## 0. 架构与端口

```
浏览器 :6080 (noVNC) → Xvnc :1（X+VNC 二合一，仅绑容器内 5901）→ openbox → LM Studio (Electron)
```

| 端口 | 用途 | 说明 |
|---|---|---|
| 6080 | noVNC 网页 | 浏览器直接操作 LM Studio GUI |
| 1234 | OpenAI 兼容 API | GUI 里需开 "Serve on Local Network" |
| 6022→22 | SSH | root/admin@123，维护用 |
| 5901 | VNC | 仅容器内 127.0.0.1，不对外 |

宿主机前提：NVIDIA 驱动 + nvidia-container-toolkit（Unraid 装 NVIDIA 驱动插件，宿主机 `nvidia-smi` 正常）。

## 1. 从 ubuntu:22.04 容器构建镜像（逐条命令）

### 1.1 起构建容器

```bash
docker run -it --name lmstudio-build ubuntu:22.04 bash
```

### 1.2 换阿里源（强制 http，避免 https 证书错误）

```bash
# 基础镜像此时还没装 ca-certificates，sources.list 若为 https 会报证书验证失败；
# 因此换域名的同时把协议强制为 http（装完 ca-certificates 后可按需切回 https）
sed -i 's|//archive.ubuntu.com|//mirrors.aliyun.com|g; s|//security.ubuntu.com|//mirrors.aliyun.com|g; s|https:|http:|g' /etc/apt/sources.list
```

### 1.3 安装全部依赖包（含 vim / openssh-server，末尾清理安装包）

```bash
apt-get update && apt-get install -y --no-install-recommends \
  vim curl ca-certificates xterm xdg-utils xdotool openssh-server \
  tigervnc-standalone-server novnc websockify openbox \
  fonts-noto-cjk \
  libgtk-3-0 libnotify4 libnss3 libxss1 libxtst6 libasound2 \
  libatk-bridge2.0-0 libdrm2 libgbm1 libx11-xcb1 libxcomposite1 \
  libxcursor1 libxdamage1 libxfixes3 libxi6 libxrandr2 libxrender1 \
  libpango-1.0-0 libpangocairo-1.0-0 libcairo2 libexpat1 \
  libfontconfig1 libnspr4 libdbus-1-3 libxkbcommon0 libcups2 \
  libgomp1 libvulkan1 libatomic1 \
  && apt-get clean && rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*
```

包用途速查：
- 远程桌面栈：tigervnc-standalone-server（Xvnc）、novnc、websockify、openbox、xterm
- 字体：fonts-noto-cjk（GUI 中文显示必备，缺了全是方框）
- Electron 运行库：libgtk-3-0 到 libcups2 那一大段
- **libatomic1：LM Studio 原生辅助进程依赖，缺了全部 exit 127（.253 实锤坑）**
- libgomp1（CUDA 后端 OpenMP 依赖）、libvulkan1（硬件检测）
- 工具：vim、openssh-server、curl、xdotool（F11 全屏）

⚠️ 本清单仅适用 22.04；24.04 里 libasound2/libcups2 改名 libasound2t64/libcups2t64。

### 1.4 Locale（英文环境下顺利编辑和显示中文）

⚠️ 实测经验：基础镜像没有 `locale-gen`（属 locales 包，未装时命令不存在）；而 en_US.UTF-8 未生成时，`LANG=en_US.UTF-8` 是无效值（`locale` 命令报 `Cannot set LC_*` 错误）。

容器内最稳做法：用 glibc 内置的 **C.UTF-8**（`locale -a` 自带，无需生成任何 locale）：

```bash
# /etc/environment 只写 LANG，不写 LC_ALL（LC_ALL 优先级最高，不适合写在这里）
echo 'LANG=C.UTF-8' >> /etc/environment

# 若之前写入过无效的 LANG=en_US.UTF-8（老文档遗留），替换掉：
sed -i 's/^LANG=en_US.UTF-8/LANG=C.UTF-8/' /etc/environment
```

- SSH 登录会话继承该值，vim 编辑含中文注释的脚本不乱码
- GUI 中文显示靠 fonts-noto-cjk，与 locale 无关
- start.sh 里显式 `export LANG=C.UTF-8`

### 1.5 SSH（root 密码登录）

```bash
echo 'root:admin@123' | chpasswd
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
mkdir -p /var/run/sshd
```

Ubuntu 默认 `PermitRootLogin prohibit-password`，不改成 yes 无法密码登录。仅限内网使用。

### 1.6 下载安装 LM Studio 0.4.24-1

**路径 A（固定版本 URL，推荐）：**
```bash
curl -fSL -o /tmp/LM-Studio-0.4.24-1-x64.deb \
  https://github.com/udit-001/lmstudio-linux-release/releases/download/v0.4.24-1/LM-Studio-0.4.24-1-x64.deb
```
**路径 B（官方 latest 重定向，无固定版本号）：**
```bash
curl -fSL -o /tmp/lmstudio.deb "https://lmstudio.ai/download/latest/linux/x64?format=deb"
```
**路径 C（.253 GitHub 直连不通时）：** 在 .196 下载后 `scp` 到 .253 宿主挂载目录，容器内直接装。

```bash
apt-get install -y /tmp/LM-Studio-0.4.24-1-x64.deb   # apt 自动补齐 .deb 其余依赖
rm -f /tmp/LM-Studio-*.deb
```

### 1.7 start.sh（放 /lmstudio/start.sh）

```bash
mkdir -p /lmstudio/logs
vim /lmstudio/start.sh    # vim 里先 :set paste 再粘贴，防自动缩进错乱
chmod +x /lmstudio/start.sh
```

脚本全文见本目录 `start.sh`（与容器内 /lmstudio/start.sh 同一版本）：
- UTF-8 locale 显式 export
- sshd 启动（供 6022 维护入口）
- Xvnc（5901 仅容器内）→ openbox → websockify（6080↔5901）→ lm-studio `--no-sandbox --user-data-dir="/lmstudio/"`
- 日志统一 /lmstudio/logs/
- 守护循环：Xvnc 挂了退出容器，交给 Docker 重启策略

### 1.8 noVNC 默认页（Unraid WebUI 直达）

noVNC 目录默认没有 index.html，直接访问 `http://<IP>:<noVNC端口>/` 会 404。把它指向 vnc.html 后，Unraid WebUI 配置才能一点就开：

```bash
# start.sh 已内置等效步骤（ln -sf vnc.html index.html）；现有容器可手动执行一次：
cd /usr/share/novnc && cp -f vnc.html index.html
```

### 1.9 提交为镜像

```bash
exit
docker commit lmstudio-build lmstudio-novnc:0.4.24
docker rm lmstudio-build
```

## 2. docker run 完整命令（生产）

```bash
docker run -d --name lmstudio \
  --gpus all \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  --ulimit memlock=-1 \
  --shm-size=2g \
  --security-opt seccomp=unconfined \
  -p 6080:6080 \
  -p 1234:1234 \
  -p 6022:22 \
  -v /mnt/disks/nvme/docker/lmstudio:/lmstudio \
  --restart unless-stopped \
  lmstudio-novnc:0.4.24
```

| 参数 | 作用 | 缺了会怎样 |
|---|---|---|
| `--gpus all` | GPU 注入主体 | 看不到 A5000 |
| `NVIDIA_DRIVER_CAPABILITIES=all` | 注入全部驱动库（compute/utility/graphics） | 默认仅 compute+utility，检测不全 |
| `--ulimit memlock=-1` | CUDA pinned memory | 大模型加载失败或性能暴跌 |
| `--shm-size=2g` | /dev/shm 上限 | 默认 64MB，Chromium 渲染进程崩 |
| `--security-opt seccomp=unconfined` | 放行 Chromium 系统调用 | 可能启动异常 |
| `-v ...:/lmstudio` | 模型/配置/日志持久化 | 容器一删全没 |

## 3. 验证清单

1. `docker exec lmstudio nvidia-smi` → 看到 RTX A5000
2. 浏览器 `http://<IP>:6080/vnc.html?autoconnect=1` → LM Studio GUI 弹出
3. 首次：欢迎向导会联网下载推理运行时（几百 MB，需外网）
4. Settings → Hardware → 出现 RTX A5000（若仍没有：`docker exec lmstudio ldd /opt/LM-Studio/resources/app/.webpack/bin/node | grep "not found"`，缺库就补装）
5. 建议在 GUI 里把 My Models 目录设到 `/lmstudio/models`（确保模型落在挂载卷里）
6. Developer → Start Server + **Serve on Local Network**
7. `curl http://<IP>:1234/v1/models` → 返回模型列表

## 4. Unraid Web UI 部署说明

### 4.1 前提
- Community Applications 安装 **NVIDIA Driver 插件**，宿主机 `nvidia-smi` 正常
- Docker 服务开启（Settings → Docker）

### 4.2 镜像进 Unraid（二选一）
- **A. Unraid 终端直接构建**：Web 终端里执行第 1 节全部命令（起 ubuntu:22.04 → 装包 → commit 成 lmstudio-novnc:0.4.24）
- **B. 外部构建导入**：有 Dockerfile 的机器 `docker save lmstudio-novnc:0.4.24 | ssh root@192.168.254.253 'docker load'`

### 4.3 Docker 页 → Add Container 模板参数

| 模板项 | 值 |
|---|---|
| Name | lmstudio |
| Repository | lmstudio-novnc:0.4.24 |
| Network Type | bridge |
| Console Shell Command | 默认 |
| WebUI | http://[IP]:[PORT:60007]/ |
| Port 1 | 6080 → 6080 (TCP) |
| Port 2 | 1234 → 1234 (TCP) |
| Port 3 | 6022 → 22 (TCP) |
| Path | /mnt/disks/nvme/docker/lmstudio → /lmstudio |
| Variable | Key=NVIDIA_VISIBLE_DEVICES, Value=all |
| Variable | Key=NVIDIA_DRIVER_CAPABILITIES, Value=all |
| Extra Parameters | `--gpus all --shm-size=2g --ulimit memlock=-1 --security-opt seccomp=unconfined --restart unless-stopped` |

（如模板有 NVIDIA 显卡下拉框，选 GPU-xxx 或 all，等效 NVIDIA_VISIBLE_DEVICES）

WebUI 说明：`[PORT:xxxx]` 按端口映射关系填，Unraid 会解析成宿主机实际端口，实测 `http://[IP]:[PORT:60007]` 点击直达 noVNC。前提是 noVNC 默认页已配置（见 1.8）；想免点击自动连接可配 `http://[IP]:[PORT:6080]/vnc.html?autoconnect=1`。

### 4.3.1 生产实机 docker run 命令（Unraid 模板导出）

Unraid Docker 模板保存后可导出等价 docker run 命令，生产实机（X399）当前在用的完整参数如下（可直接用于命令行部署或重建）：

```bash
docker run \
  -d \
  --name='LMStudio' \
  --net='bridge' \
  --pids-limit 65536 \
  -e TZ="Asia/Shanghai" \
  -e HOST_OS="Unraid" \
  -e HOST_HOSTNAME="X399" \
  -e HOST_CONTAINERNAME="LMStudio" \
  -e 'NVIDIA_VISIBLE_DEVICES'='all' \
  -e 'NVIDIA_DRIVER_CAPABILITIES'='all' \
  -l net.unraid.docker.managed=dockerman \
  -l net.unraid.docker.webui='http://[IP]:[PORT:60007]' \
  -l net.unraid.docker.icon='https://lmstudio.ai/assets/marketing/lmstudio-app-logo.webp' \
  -p '6022:22/tcp' \
  -p '1234:1234/tcp' \
  -p '60007:6080/tcp' \
  -v '/mnt/disks/nvme/docker/app/lmstudio/':'/lmstudio':'rw' \
  --gpus all \
  --shm-size=2g \
  --ulimit memlock=-1 \
  --security-opt seccomp=unconfined \
  --restart unless-stopped \
  'lms:v1.0' \
  /lmstudio/start.sh
```

要点：

- 镜像名 `lms:v1.0` 为生产实际命名（等价于手册里 commit 的 lmstudio-novnc，名字自定，模板/命令保持一致即可）
- 末尾 `/lmstudio/start.sh` 显式指定启动命令；若镜像构建时 ENTRYPOINT 已是它，可省略
- 端口映射 `60007:6080`：宿主 60007 → 容器 6080（noVNC），WebUI 标签里的 `[PORT:60007]` 与之对应
- `--pids-limit 65536`：Unraid 模板默认值，防容器内进程数失控，保留
- `HOST_*` 环境变量与 `net.unraid.docker.*` 标签是 Unraid 管理元数据，手工部署可省略，Unraid 模板导入时靠它们识别

### 4.4 Apply 启动后
同第 3 节验证清单（WebUI 按钮/浏览器进 6080）。

### 4.5 Compose Manager 方式（备选）
有 Compose Manager 插件时可用 include 包装器（RAGFlow 同款机制）：
`/boot/config/plugins/compose.manager/projects/lmstudio/docker-compose.yml` →
`name: lmstudio` + `include: 本目录 docker-compose.yml 的宿主路径`。
compose 已含 gpus/env/ulimits/端口/挂载全套，`docker compose config --services` 验证后 UI 可启停。

## 5. 已知坑速查

| 坑 | 现象 | 处理 |
|---|---|---|
| 换源后 https 报证书错误 | `apt-get update` 报 Certificate verification failed | 换源时强制 http：`sed -i 's/https:/http:/g' /etc/apt/sources.list`（基础镜像无 ca-certificates，装完再切 https） |
| 缺 libatomic1 | 所有原生子进程 exit 127，GUI 无 GPU | `apt-get install -y libatomic1` + 重启 |
| .253 GitHub 不通 | curl 下载 deb 失败 | .196 下载后 scp 中转，或走官方 latest 重定向 |
| GPU 显存被占 | A5000 只剩部分可用 | 宿主机 `nvidia-smi` 查占用容器（如 RAGFlow GPU），错峰或调小 offload |
| Serve on Local Network 未开 | 1234 映射了但 API 不通 | GUI Developer 里打开开关 |
| Vulkan 检测无 NVIDIA | Hardware 页 Vulkan 项空 | 容器无 nvidia_icd.json，不影响 CUDA 推理，忽略 |
| 模型下载慢 | HuggingFace 拉不动 | 本地下 GGUF 放 /lmstudio/models，GUI 里改 My Models 目录 |
