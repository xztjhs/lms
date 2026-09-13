# lms — LM Studio + noVNC 容器镜像

Ubuntu 22.04 容器化的 LM Studio 0.4.24：浏览器直接操作完整桌面 GUI，支持 NVIDIA GPU 推理。
镜像：`ghcr.io/xztjhs/lms:v1.0`（private）

## 架构

```
浏览器 → noVNC (:6080) → Xvnc :1（X+VNC 二合一）→ openbox → LM Studio（Electron）
另暴露 :1234 OpenAI 兼容 API，:22 SSH 维护入口
```

## 特性

- LM Studio 0.4.24-1 官方 .deb 安装，Electron 以 `--no-sandbox` 运行
- Xvnc（tigervnc）替代 Xvfb+x11vnc 双进程方案，栈更短、故障面更小
- noVNC 默认页 + autoconnect，网页即开即用
- NVIDIA GPU 透传（`--gpus all`，实测 RTX A5000 24G）；CUDA 推理运行时首次加载模型时自动下载
- 内置 openssh-server，维护不依赖桌面
- 持久化目录 `/lmstudio`：模型、配置、日志全在一处

## 快速开始

```bash
docker run -d --name lmstudio \
  --gpus all \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  --ulimit memlock=-1 \
  --shm-size=2g \
  --security-opt seccomp=unconfined \
  -p 6080:6080 -p 1234:1234 -p 6022:22 \
  -v /mnt/disks/nvme/docker/app/lmstudio:/lmstudio \
  --restart unless-stopped \
  ghcr.io/xztjhs/lms:v1.0
```

打开 `http://<IP>:6080/vnc.html?autoconnect=1` 即可操作 LM Studio 完整界面。

## 踩坑精华（完整手册见 [DEPLOY.md](DEPLOY.md)）

- **libatomic1 必装**：缺它 LM Studio 所有原生子进程 exit 127，硬件检测全灭，GUI 看不到 GPU
- **换阿里源强制 http**：基础镜像没有 ca-certificates，https 源报证书错误
- **locale 用内置 C.UTF-8**：无需 locale-gen；en_US.UTF-8 未生成时 LANG 值无效
- **API 需在 GUI 开 Serve on Local Network**：否则 1234 只监听容器内 127.0.0.1
- 端口映射采用 Unraid 生产配置：60007:6080 / 1234 / 6022:22

## 安全

镜像 `/etc/shadow` 含 root 密码哈希：**仅限私有使用，勿设为 public**。

## 版本

| 日期 | 版本 | 说明 |
|---|---|---|
| 2026-09-12 | v1.0 | 首版：LM Studio 0.4.24-1 / Ubuntu 22.04 / GPU 透传 / 实排调优（GPU 检测修复） |
