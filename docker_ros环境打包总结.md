# Docker 管理 ROS/CUDA 运行环境总结

生成时间：2026-04-23

来源：本文件是根据本次 Codex 会话整理出的总结，不是聊天原文备份。会话原始 JSONL 位于：

```text
/home/james/.codex/sessions/2026/04/20/rollout-2026-04-20T20-32-28-019daae0-bd7b-7bc0-a9e9-326ef013ac84.jsonl
```

## 1. 最终目标

你的目标不是把一台 Ubuntu 机器 100% 无依赖复制到另一台机器，而是把复杂运行环境尽量封装进 Docker 镜像里。

更准确的目标是：

```text
新电脑先安装最少宿主机环境：
Ubuntu + NVIDIA 显卡驱动 + Docker + NVIDIA Container Toolkit + 必要 udev/网络/远程控制

然后导入已经配置好的 Docker 镜像：
CUDA 11.8 + cuDNN + ROS + OpenCV 4.7 + TensorRT + PCL/Fast_GICP + K4A/雷达驱动 + 项目代码

最后用 docker run/start/exec 启动容器，直接运行算法和 ROS 节点。
```

这条路线是可行的，而且比“完全自动化装机脚本”更现实。它接受一个事实：硬件驱动、内核模块和设备权限仍然属于宿主机；算法运行环境、用户态库和 ROS 工作区尽量放入容器。

## 2. 宿主机和容器的职责边界

### 2.1 必须留在宿主机的部分

这些东西和内核、物理硬件、系统服务绑定，不适合放进容器：

- NVIDIA 显卡驱动
- Linux 内核和内核模块
- Docker Engine 本体
- NVIDIA Container Toolkit
- 物理网卡配置，例如 MID360 雷达直连网卡 IP
- udev 规则，例如 Azure Kinect、串口、USB 设备权限
- 远程控制软件，例如 SSH、RustDesk
- NetworkManager、GNOME control-center 等宿主机系统设置

### 2.2 适合放进容器的部分

这些是用户态环境，可以放入镜像：

- Ubuntu 20.04 用户态
- CUDA 11.8 用户态库
- cuDNN 8
- ROS 本体，例如 ROS1 Noetic
- OpenCV 4.7 和 opencv_contrib
- TensorRT 8.6.1.6
- PCL、Fast_GICP
- Kinect SDK 用户态包，例如 `k4a-tools`、`libk4a`
- Livox SDK / 雷达 ROS 驱动
- Conda、pip、Python 包、PyTorch
- 项目代码、ROS 工作区、编译产物

### 2.3 推荐架构

```text
宿主机 Ubuntu 22.04 或 20.04
  ├─ NVIDIA driver
  ├─ Docker
  ├─ NVIDIA Container Toolkit
  ├─ SSH / RustDesk
  ├─ udev rules
  └─ 网卡 IP 配置

Docker 容器 Ubuntu 20.04
  ├─ CUDA 11.8 + cuDNN 8
  ├─ ROS
  ├─ OpenCV 4.7
  ├─ TensorRT
  ├─ PCL / Fast_GICP
  ├─ K4A / Livox SDK
  └─ 项目工作区
```

## 3. 宿主机可以是 Ubuntu 22.04，容器可以是 Ubuntu 20.04

这是可行的。

Docker 容器不是完整虚拟机。容器有独立的用户态文件系统，但共享宿主机内核。因此：

```text
宿主机：Ubuntu 22.04
容器内：Ubuntu 20.04
内核：仍然是宿主机的内核
```

这对 ROS1 Noetic 很有用。Noetic 主要对应 Ubuntu 20.04，如果宿主机想升级到 22.04，仍然可以让容器保持 20.04。

要注意的是：容器里看到的是 Ubuntu 20.04 用户态，但不是一台完整 Ubuntu 20.04 虚拟机。凡是依赖宿主机内核、真实硬件、NetworkManager 的东西，仍然在宿主机配置。

## 4. Docker 基础概念

### 4.1 镜像 image

镜像是模板、快照、系统安装包。

例子：

```bash
nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04
```

这不是正在运行的系统，而是一个可以启动容器的模板。

可以理解为：

```text
镜像 = 系统模板
```

### 4.2 容器 container

容器是从镜像启动出来的运行实例。

可以理解为：

```text
容器 = 从模板启动出来的系统实例
```

同一个镜像可以启动多个容器。你在容器里安装软件、修改文件，这些改动发生在容器内部，不会自动变成新镜像。要保存成镜像，需要 `docker commit`。

### 4.3 镜像层 layer

Docker 镜像由多层叠起来。比如你的环境大概是：

```text
nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04
  └─ mylab:ros-cuda-user
       └─ my_lab_k4a:latest
```

每次 `docker commit` 都会基于当前容器生成一个新的镜像层。

### 4.4 基础缓存

“基础缓存”不是 Docker 官方术语，是这里为了方便理解说的。

意思是：先把很大的基础镜像下载到本机，后面都基于它做环境。比如：

```bash
sudo docker pull nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04
```

这个镜像里已经有：

- Ubuntu 20.04
- CUDA 11.8
- cuDNN 8
- CUDA 开发工具，例如 `nvcc`
- NVIDIA 相关开发库

拉下来一次后，本机就缓存了它。后续 `docker run` 不会重新下载。

## 5. 为什么选择 NVIDIA CUDA 官方镜像

推荐使用：

```bash
nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04
```

不推荐从纯 `ubuntu:20.04` 手动安装 CUDA/cuDNN，原因是：

- 官方 CUDA 镜像已经配置好 CUDA/cuDNN 路径、库、环境变量。
- 你后面还要装 ROS、OpenCV、TensorRT、PCL，没有必要一开始就把 CUDA/cuDNN 变成不确定因素。
- 手动安装容易出现 `nvcc`、`LD_LIBRARY_PATH`、头文件、动态库版本不匹配。
- 镜像拉下来一次后可以 `docker save`，以后不需要每台机器重新下载。

为什么用 `devel` 而不是 `runtime`：

```text
runtime：只适合运行 CUDA 程序
devel：包含编译 CUDA 程序所需的 nvcc、headers、开发库
```

你后面要编译 OpenCV CUDA、TensorRT 相关代码，推荐用 `devel`。

## 6. Docker 安装和早期问题总结

### 6.1 USTC Docker GPG 源 TLS 失败

遇到过：

```text
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to mirrors.ustc.edu.cn:443
```

原因不是 Docker 本身，而是 `curl` 到 USTC 镜像站的 HTTPS/TLS 链路被中断。后来日志显示请求走了代理：

```text
HTTPS_PROXY == http://127.0.0.1:7890
```

同一代理访问 Docker 官方源成功，说明：

- 本机 `curl` 没坏
- CA 证书没坏
- Docker 官方源可用
- 主要是代理访问 USTC 镜像站不稳定

解决思路：

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

如果国内源通过代理出问题，优先换源或不让国内源走代理。

### 6.2 docker.sock permission denied

遇到过：

```text
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

原因：当前用户没有权限访问 Docker socket。

临时解决：

```bash
sudo docker run ...
```

长期解决：

```bash
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker $USER
newgrp docker
```

或者重新登录。

确认：

```bash
id
ls -l /var/run/docker.sock
```

注意：`sudo systemctl enable docker` 只表示 Docker 开机自启，不会给当前用户增加 socket 权限。

### 6.3 Docker daemon 拉镜像超时

遇到过：

```text
Get "https://registry-1.docker.io/v2/": context deadline exceeded
```

原因：`docker pull` 真正联网的是宿主机上的 `dockerd` 服务，不是当前 shell。shell 里有代理不代表 Docker daemon 有代理。

给 Docker daemon 配代理：

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null <<'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,::1"
EOF
```

重载并重启：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

检查：

```bash
systemctl show --property=Environment docker
```

### 6.4 training/webapp 镜像过老

菜鸟教程里使用：

```bash
docker pull training/webapp
docker run -d -P training/webapp python app.py
```

这个镜像太老，属于旧格式镜像。现在 Docker 新版本默认禁用或移除了旧的 `Docker Image v1` / `manifest v2 schema 1` 支持。

结论：

```text
不是你机器坏了，是教程示例过时。
```

用现代镜像测试：

```bash
docker run --rm hello-world
docker run --rm ubuntu:20.04 /bin/echo "Hello World"
docker run -d --name mynginx -p 8080:80 nginx:alpine
```

## 7. NVIDIA Container Toolkit

### 7.1 它是什么

NVIDIA Container Toolkit 是 Docker 和 NVIDIA 驱动之间的桥接工具。

它不是：

- 显卡驱动
- CUDA 本体
- 一个容器镜像

它的作用是：

- 让容器看到宿主机 GPU
- 把宿主机 NVIDIA 设备节点和驱动能力暴露给容器
- 支持 `docker run --gpus all ...`

关系可以理解为：

```text
NVIDIA 驱动：让宿主机显卡正常工作
Docker：运行容器
NVIDIA Container Toolkit：让 Docker 把 GPU 交给容器
CUDA 容器镜像：容器里的 CUDA 用户态环境
```

### 7.2 安装和配置

先确认宿主机驱动：

```bash
nvidia-smi
```

安装 Toolkit 后配置 Docker runtime：

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

验证：

```bash
sudo docker run --rm --gpus all nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04 nvidia-smi
```

如果能在容器里看到显卡信息，说明：

```text
宿主机 NVIDIA 驱动 OK
Docker OK
NVIDIA Container Toolkit OK
CUDA 11.8 容器能使用 GPU
```

### 7.3 常见错误

错误：

```text
failed to discover GPU vendor from CDI: no known GPU vendor found
```

可能原因：

- 宿主机没有 NVIDIA 显卡
- 宿主机 `nvidia-smi` 不正常
- NVIDIA Container Toolkit 未安装
- Toolkit 安装了但没有 `nvidia-ctk runtime configure`
- CDI 配置异常

错误：

```text
AMD CDI spec not found
```

这类错误也和 Docker GPU 设备发现/CDI 配置有关。最终你已经通过安装和配置 Toolkit 跑通了 `nvidia-smi`。

## 8. 推荐的容器创建命令

### 8.1 第一次创建长期容器

这个命令只在创建容器时执行一次：

```bash
xhost +local:

sudo docker run -it --name ros_cuda_env_user \
  --gpus all \
  --network host \
  --ipc host \
  --privileged \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /dev:/dev \
  -v /dev/bus/usb:/dev/bus/usb \
  -v /run/udev:/run/udev:ro \
  -v /etc/localtime:/etc/localtime:ro \
  nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04 \
  bash
```

如果已经有阶段镜像，把最后的基础镜像替换成你的镜像：

```bash
my_lab_k4a:latest
```

### 8.2 参数解释

`sudo docker run`

创建并启动一个新容器。注意：`docker run` 是“新建容器”，不是“进入已有容器”。

`-it`

交互式终端。

- `-i` 保持标准输入打开
- `-t` 分配伪终端

没有 `-it`，你就不能像正常终端一样输入命令。

`--name ros_cuda_env_user`

给容器起名字。以后可以用这个名字 `start`、`exec`、`commit`。

如果名字已存在，再次 `docker run --name` 会失败。已有容器应该用 `docker start`。

`--gpus all`

把宿主机所有 NVIDIA GPU 暴露给容器。

依赖：

- 宿主机 NVIDIA 驱动正常
- NVIDIA Container Toolkit 正常

`--network host`

容器使用宿主机网络命名空间。

效果：

- 容器没有独立局域网 IP
- 容器直接使用宿主机 IP
- 容器里开的端口就是宿主机上的端口
- 对 ROS、雷达、UDP 传感器、局域网设备最省事

代价：

- 网络隔离弱
- 容器和宿主机端口可能冲突

`--ipc host`

容器共享宿主机 IPC 命名空间。

对一些 GUI、共享内存、ROS、图像处理场景更省事。

`--privileged`

给容器较高权限。

对调试硬件很方便：

- USB 相机
- 雷达
- 串口
- `/dev` 设备

代价是安全性下降。实验室本机调试可以先用它跑通，后期再考虑收紧权限。

`-e DISPLAY=$DISPLAY`

把宿主机的 `DISPLAY` 环境变量传给容器。容器里的 GUI 程序用它知道窗口显示到哪里。

`-e QT_X11_NO_MITSHM=1`

避免 Qt 程序在 Docker/X11 环境中因为 MIT-SHM 共享内存出现显示问题。对 `rviz`、Qt GUI 有帮助。

`-v /tmp/.X11-unix:/tmp/.X11-unix`

把宿主机 X11 socket 挂进容器。GUI 程序能显示到宿主机桌面主要靠这个和 `DISPLAY`。

`-v /dev:/dev`

把宿主机设备目录映射进容器。调试 USB、相机、雷达、串口时常用。

`-v /dev/bus/usb:/dev/bus/usb`

显式映射 USB 总线。Azure Kinect 这类基于 libusb 的设备尤其需要。

`-v /run/udev:/run/udev:ro`

把宿主机 udev 运行时信息只读映射进容器。某些 SDK 会通过 udev 信息识别 USB 设备。

`:ro` 表示 read-only，只读挂载。

`-v /etc/localtime:/etc/localtime:ro`

让容器时间和宿主机一致。

`nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04`

使用的镜像。

`bash`

容器启动后执行的命令，这里是进入 bash。

## 9. 每天进入容器

创建好容器后，不需要每天写长命令。

启动已有容器：

```bash
sudo docker start ros_cuda_env_user
```

进入已有容器：

```bash
sudo docker exec -it -u gugugaga -w /home/gugugaga ros_cuda_env_user bash
```

推荐别名：

```bash
alias denter='sudo docker start ros_cuda_env_user >/dev/null && sudo docker exec -it -u gugugaga -w /home/gugugaga ros_cuda_env_user bash'
```

如果宿主机是 zsh，写进：

```bash
~/.zshrc
```

如果宿主机是 bash，写进：

```bash
~/.bashrc
```

然后执行：

```bash
source ~/.zshrc
```

或：

```bash
source ~/.bashrc
```

之后进入容器：

```bash
denter
```

## 10. docker run、start、exec 的区别

`docker run`

创建一个新容器并启动。

```bash
docker run -it --name my_container image_name bash
```

只在创建容器时用。创建后，网络模式、挂载目录、设备映射等参数基本固定。

`docker start`

启动已经存在的容器。

```bash
docker start ros_cuda_env_user
```

`docker exec`

在已经运行的容器里执行一个新命令，常用于进入 shell。

```bash
docker exec -it ros_cuda_env_user bash
```

## 11. --rm 参数详解

`--rm` 的意思是：容器运行结束后自动删除容器。

适合临时测试：

```bash
sudo docker run --rm hello-world
sudo docker run --rm --gpus all nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04 nvidia-smi
```

这类命令只是执行一次测试，执行完容器就没用了。加 `--rm` 可以避免留下垃圾容器。

不适合长期环境：

```bash
sudo docker run --rm -it --name ros_cuda_env_user ... bash
```

如果你在这个容器里安装 ROS、OpenCV、TensorRT，然后退出，容器会被自动删除。除非你已经 `commit`，否则安装内容会丢。

结论：

```text
临时测试用 --rm
长期手工装环境不要用 --rm
```

## 12. 容器普通用户

基础镜像默认通常是 root。你创建了普通用户：

```bash
useradd -m -s /bin/bash gugugaga
passwd gugugaga
```

安装 sudo：

```bash
apt update
apt install -y sudo bash-completion vim nano git
```

给用户 sudo 权限：

```bash
usermod -aG sudo gugugaga
```

免密 sudo：

```bash
echo 'gugugaga ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/gugugaga
chmod 0440 /etc/sudoers.d/gugugaga
```

当时犯过一个小错：用户叫 `gugugaga`，但 sudoers 写成了 `dev`：

```bash
echo 'dev ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/dev
```

这不会给 `gugugaga` 权限。应该删除错误文件：

```bash
rm -f /etc/sudoers.d/dev
```

测试：

```bash
su - gugugaga
whoami
sudo whoami
```

预期：

```text
gugugaga
root
```

## 13. 让容器默认用户变成普通用户

已有容器默认进 root，是因为镜像默认用户是 root。

可以在 commit 时设置默认用户：

```bash
sudo docker commit \
  --change 'USER gugugaga' \
  --change 'WORKDIR /home/gugugaga' \
  ros_cuda_env_user \
  mylab:ros-cuda-user
```

之后用新镜像创建容器，默认就是 `gugugaga`。

不过在环境还没装完时，建议先用 alias 指定用户。因为安装系统软件时 root 更方便。

## 14. 终端颜色和 bash 配置

容器里终端全白是正常的，原因可能是：

- 没有彩色 PS1
- 没有 `ls --color=auto`
- 没有加载 Ubuntu 默认 `.bashrc`
- 容器是最小环境，不像实体机那样装了 zsh/fish/主题

可以给普通用户配置：

```bash
cat >> ~/.bashrc <<'EOF'

force_color_prompt=yes

if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
    alias ls='ls --color=auto'
    alias grep='grep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias egrep='egrep --color=auto'
fi

if [ "$color_prompt" = yes ] || [ "$force_color_prompt" = yes ]; then
    PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
else
    PS1='\u@\h:\w\$ '
fi

alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
EOF
```

生效：

```bash
source ~/.bashrc
```

## 15. 图形界面 GUI

容器里的 GUI 程序可以显示到宿主机桌面，例如：

- `rviz`
- `rqt`
- `k4aviewer`
- OpenCV `imshow`
- Qt 程序
- GTK 程序

这不是“容器自己开桌面”，而是“容器里的 GUI 程序把窗口显示到宿主机 X11 桌面”。

宿主机先执行：

```bash
xhost +local:
```

创建容器时带：

```bash
-e DISPLAY=$DISPLAY
-e QT_X11_NO_MITSHM=1
-v /tmp/.X11-unix:/tmp/.X11-unix
```

测试：

```bash
sudo apt install -y x11-apps
xeyes
```

如果弹出眼睛窗口，GUI 基本通了。

## 16. 不建议在容器里启动 terminator 作为入口

你曾经使用：

```bash
alias denter='sudo docker start ros_cuda_env_user >/dev/null && sudo docker exec -it ros_cuda_env_user terminator'
```

现象：

```text
宿主机 terminator 不退出
关闭宿主机 terminator 后，容器里的 terminator 也关闭
```

原因：你不是“进入 Docker”，而是在宿主终端里前台执行了容器内 GUI 程序 `terminator`。宿主终端会等待这个 `docker exec` 进程结束。

警告：

```text
Couldn't connect to accessibility bus
```

一般是容器没有宿主机无障碍服务总线，可忽略。

警告：

```text
Binding '<Control><Alt>a' failed
```

宿主机 Terminator 和容器 Terminator 抢同一个快捷键。

推荐：

```bash
alias denter='sudo docker start ros_cuda_env_user >/dev/null && sudo docker exec -it -u gugugaga -w /home/gugugaga ros_cuda_env_user bash'
```

也就是：宿主机用自己的 Terminator，里面运行容器 bash。

## 17. SSH 远程连接

推荐做法：

```text
远程 SSH 到宿主机
然后在宿主机 docker exec 进入容器
```

流程：

```bash
ssh l@宿主机IP
denter
```

不推荐在容器里单独跑 SSH 服务，原因：

- 容器不是完整虚拟机
- 容器默认不跑完整 systemd
- 如果 `--network host`，容器和宿主机共用 IP，22 端口容易冲突
- 远程、安全、网络交给宿主机更清晰

如果必须在容器里开 SSH，建议避开 22 端口，例如 2222。但当前方案不需要。

## 18. 代理配置

Docker 场景里有三层代理，不能混为一谈。

### 18.1 docker pull 的代理

`docker pull` 是宿主机 Docker daemon 联网。

配置位置：

```text
/etc/systemd/system/docker.service.d/http-proxy.conf
```

检查：

```bash
systemctl show --property=Environment docker
```

### 18.2 容器内命令的代理

容器里运行：

```bash
apt update
pip install
git clone
curl
wget
```

这些用的是容器内环境变量。

如果容器使用 `--network host`，容器可以直接访问宿主机的 `127.0.0.1:7890`：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
export no_proxy=localhost,127.0.0.1,::1
export NO_PROXY=localhost,127.0.0.1,::1
```

如果不使用 `--network host`，容器里的 `127.0.0.1` 是容器自己，不是宿主机。通常要用 Docker 网关地址，例如 `172.17.0.1`。

### 18.3 apt 代理

apt 有时需要单独配置：

```bash
sudo tee /etc/apt/apt.conf.d/99proxy > /dev/null <<'EOF'
Acquire::http::Proxy "http://127.0.0.1:7890";
Acquire::https::Proxy "http://127.0.0.1:7890";
EOF
```

删除：

```bash
sudo rm /etc/apt/apt.conf.d/99proxy
```

### 18.4 国内源不要盲目走代理

你遇到过：

```text
502 Bad Gateway [IP: 127.0.0.1 7890]
```

这是容器里的 apt 通过代理访问 USTC 源，代理返回 502。

处理：

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
sudo mv /etc/apt/apt.conf.d/99proxy /etc/apt/apt.conf.d/99proxy.bak 2>/dev/null || true
sudo apt update
sudo apt install --fix-missing
```

或者换阿里云源：

```bash
sudo sed -i 's#http://mirrors.ustc.edu.cn/ubuntu#http://mirrors.aliyun.com/ubuntu#g' /etc/apt/sources.list
sudo apt update
```

结论：

```text
GitHub / Docker Hub / NVIDIA GitHub Pages 可能需要代理
国内 apt 源通常直连更稳
```

## 19. GitHub clone 和 SSH 问题

HTTPS clone 报错：

```text
gnutls_handshake() failed: The TLS connection was non-properly terminated
```

原因通常是容器内访问 GitHub 的网络/代理不稳定。

解决：

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

取消：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

SSH 测试：

```bash
ssh -vT git@github.com
```

你遇到过卡在：

```text
Connecting to github.com [20.205.243.166] port 22.
Connection established.
Local version string SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.13
```

这说明 TCP 22 已连接，但 SSH 后续握手不继续。不是密钥问题，而是链路被干扰或不稳定。

建议改用 GitHub SSH over 443：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat > ~/.ssh/config <<'EOF'
Host github.com
    HostName ssh.github.com
    User git
    Port 443
    ServerAliveInterval 30
    ServerAliveCountMax 6
EOF
chmod 600 ~/.ssh/config
```

再测：

```bash
ssh -T git@github.com
```

如果只是 clone 公共仓库，HTTPS + 代理更省事。

## 20. ROS 安装策略

最终决定：ROS 本体也装进容器。

原因：

- 容器里编译/运行 ROS 包需要完整 ROS 环境。
- 只把 ROS 依赖放进容器、ROS 本体放宿主机会导致路径、消息、库版本混乱。
- ROS、CUDA、OpenCV、TensorRT、项目代码都在容器里，边界最清晰。

可以用 FishROS 一键安装，但建议在普通用户下跑：

```bash
sudo docker exec -it -u gugugaga -w /home/gugugaga ros_cuda_env_user bash
```

容器里先装基础工具：

```bash
sudo apt update
sudo apt install -y wget curl gnupg2 lsb-release ca-certificates python3-yaml locales
```

locale：

```bash
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```

FishROS：

```bash
wget http://fishros.com/install -O fishros
bash fishros
```

安装完：

```bash
source ~/.bashrc
printenv | grep ROS
roscore
```

装好 ROS 后建议立刻阶段保存：

```bash
sudo docker commit ros_cuda_env_user mylab:ros-noetic-cuda118-stage1
```

## 21. TensorRT 安装策略

当前容器已经有 CUDA 11.8、cuDNN、Ubuntu 20.04、ROS，因此不建议换 TensorRT 官方镜像。

推荐直接在当前容器里装：

```text
TensorRT 8.6.1.6 for CUDA 11.8
```

推荐 tar 包方式，而不是随意 `apt install tensorrt`：

- 版本可控
- 不容易污染系统包
- 适合 legacy 环境
- commit 后直接带走

宿主机拷贝 tar 包进容器：

```bash
sudo docker cp TensorRT-8.6.1.6.Linux.x86_64-gnu.cuda-11.8.tar.gz ros_cuda_env_user:/tmp/
```

容器里：

```bash
cd /tmp
sudo tar -xzf TensorRT-8.6.1.6.Linux.x86_64-gnu.cuda-11.8.tar.gz -C /usr/local
sudo ln -sfn /usr/local/TensorRT-8.6.1.6 /usr/local/tensorrt
```

动态库路径：

```bash
sudo tee /etc/ld.so.conf.d/tensorrt.conf > /dev/null <<'EOF'
/usr/local/tensorrt/lib
/usr/local/tensorrt/targets/x86_64-linux-gnu/lib
EOF

sudo ldconfig
```

环境变量：

```bash
sudo tee /etc/profile.d/tensorrt.sh > /dev/null <<'EOF'
export TENSORRT_ROOT=/usr/local/tensorrt
export PATH=$TENSORRT_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$TENSORRT_ROOT/lib:$TENSORRT_ROOT/targets/x86_64-linux-gnu/lib:$LD_LIBRARY_PATH
EOF

source /etc/profile.d/tensorrt.sh
```

验证：

```bash
trtexec --version
```

Python wheel：

```bash
ls /usr/local/tensorrt/python/
python -m pip install /usr/local/tensorrt/python/tensorrt-*-cp39-*.whl
```

`cp39` 要换成你的 Python 版本对应 wheel，例如 Python 3.8 用 `cp38`。

验证：

```bash
python - <<'PY'
import tensorrt as trt
print(trt.__version__)
PY
```

## 22. OpenCV、源码、缓存和镜像大小

如果你在容器里编译 OpenCV 4.7，编译完成后要考虑清理：

- OpenCV 源码目录
- opencv_contrib 源码目录
- `build/` 编译目录
- 下载的压缩包
- apt 缓存
- pip/conda 缓存

原因：你会用 `docker commit` 保存容器。commit 前没删的文件会进入新镜像。

常用清理：

```bash
sudo apt clean
sudo apt autoremove -y
sudo rm -rf /var/lib/apt/lists/*
sudo rm -rf /tmp/*
python3 -m pip cache purge 2>/dev/null || true
conda clean -a -y
```

TensorRT 压缩包装完可以删：

```bash
rm -f ~/TensorRT-*.tar.gz
rm -f /tmp/TensorRT-*.tar.gz
```

OpenCV build 目录一般可以删：

```bash
rm -rf ~/opencv-4.7.0/build
rm -rf ~/opencv_contrib-4.7.0/build
```

如果确认不再重新编译，也可以删源码：

```bash
rm -rf ~/opencv-4.7.0
rm -rf ~/opencv_contrib-4.7.0
```

注意：如果将来改用 Dockerfile 分层构建，下载、安装、删除最好写在同一个 `RUN` 中。否则旧层仍然占空间。你现在是手动容器里装完再 commit，commit 前删掉通常就不会进最终镜像。

## 23. docker commit

`docker commit` 的作用：

```text
把当前容器文件系统状态保存成新镜像。
```

示例：

```bash
sudo docker commit ros_cuda_env_user my_lab_k4a:latest
```

带默认用户：

```bash
sudo docker commit \
  --change 'USER gugugaga' \
  --change 'WORKDIR /home/gugugaga' \
  ros_cuda_env_user \
  my_lab_k4a:latest
```

commit 慢是正常的，尤其环境里有：

- CUDA/cuDNN
- ROS
- OpenCV 编译产物
- TensorRT
- `.deb`
- `.tar.gz`
- apt 缓存
- 源码和 build 目录

更稳的流程：

```bash
# 容器里先停程序并清理缓存，然后 exit
exit

# 宿主机
sudo docker stop ros_cuda_env_user
sudo docker commit ros_cuda_env_user my_lab_k4a:latest
```

注意：bind mount 挂载进容器的宿主机目录一般不会进入 commit。commit 保存的是容器自己的可写层。

## 24. docker save 和 docker load

`docker commit` 是把容器变成镜像。

`docker save` 是把镜像导出成 tar 文件。

导出：

```bash
sudo docker save -o my_lab_k4a_latest.tar my_lab_k4a:latest
```

拷贝到新机器后导入：

```bash
sudo docker load -i my_lab_k4a_latest.tar
```

再创建容器：

```bash
sudo docker run -it --name ros_cuda_env_user \
  --gpus all \
  --network host \
  --ipc host \
  --privileged \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /dev:/dev \
  -v /dev/bus/usb:/dev/bus/usb \
  -v /run/udev:/run/udev:ro \
  -v /etc/localtime:/etc/localtime:ro \
  my_lab_k4a:latest \
  bash
```

## 25. docker images 输出解释

你当前看到过：

```text
IMAGE                                         ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest                            f9078146db2e       25.9kB         9.49kB
my_lab_k4a:latest                             2668feac17ba       47.4GB         15.9GB
mylab:ros-cuda-user                           421d8687a400       15.4GB         5.45GB    U
nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04   28cb39688438       15.1GB         5.36GB    U
```

含义：

`IMAGE`

镜像名和标签。

`ID`

镜像 ID。

`DISK USAGE`

这个镜像当前在本机占用的磁盘空间。

`CONTENT SIZE`

镜像内容本身的大致大小。

`EXTRA` 里的 `U`

说明有容器正在使用这个镜像。

各镜像解释：

`hello-world:latest`

Docker 官方最小测试镜像，用于验证 Docker 能不能跑。

`nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04`

NVIDIA 官方基础镜像，是你的 CUDA/cuDNN/Ubuntu20.04 底座。

`mylab:ros-cuda-user`

你在基础镜像上添加普通用户、ROS 或基础工具后的阶段镜像。

`my_lab_k4a:latest`

你继续安装 K4A、OpenCV、TensorRT 等之后的重型完整环境镜像。

查看 Docker 总占用：

```bash
docker system df
```

查看某镜像历史：

```bash
docker history my_lab_k4a:latest
```

## 26. udev 规则

udev 不是 Docker 的前置条件。

你可以先：

```text
装 NVIDIA 驱动
装 Docker
装 NVIDIA Container Toolkit
创建容器
安装 ROS/CUDA/OpenCV/TensorRT
```

等接具体硬件时，再补 udev 规则。

udev 的作用：

```text
管理宿主机设备节点权限、名称和设备接入行为。
```

它属于宿主机，不属于容器。

什么时候需要：

- 普通用户打不开设备
- SDK 官方要求写规则
- 不想每次 `sudo`
- 需要固定设备权限或设备名

加载规则：

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

通常还需要拔插设备。

## 27. Azure Kinect / k4aviewer

你运行 `k4aviewer` 时看到大量 ALSA/PulseAudio 日志：

```text
PulseAudio: Unable to connect: Connection refused
```

这些大概率不是主因。它表示容器里没连好音频服务。除非你要用 Kinect 麦克风阵列，否则可以先忽略。

真正关键错误：

```text
find_libusb_device(). libusb device(s) are all unavailable.
k4a_device_open() returned failure
```

含义：K4A SDK 没能打开 Azure Kinect USB 设备。

常见原因：

- 宿主机没有识别设备
- 容器没有正确挂载 USB 总线
- 没挂 `/run/udev`
- 宿主机 udev 规则没生效
- 设备被另一个程序占用

宿主机检查：

```bash
lsusb | grep 045e
```

容器里检查：

```bash
lsusb | grep 045e
```

创建容器时建议包含：

```bash
-v /dev:/dev
-v /dev/bus/usb:/dev/bus/usb
-v /run/udev:/run/udev:ro
--privileged
```

如果当前容器创建时漏了这些挂载，不能用 `docker exec` 补。需要：

```bash
sudo docker commit ros_cuda_env_user mylab:k4a-test
sudo docker rm -f ros_cuda_env_user
```

然后用带完整挂载参数的 `docker run` 重新创建。

## 28. MID360 / Livox 雷达网络配置

不建议在容器里用 `control-center` 配网络。

原因：

```text
物理网卡、IP、路由、NetworkManager 都属于宿主机。
```

正确模式：

```text
宿主机负责连接雷达并配置网卡 IP
容器负责运行 ROS / Livox SDK / 驱动节点
```

如果容器用：

```bash
--network host
```

宿主机配好网卡后，容器里可以直接访问雷达。

示例，宿主机配置雷达直连网卡：

```bash
ip link
```

假设网卡名是 `enp3s0`：

```bash
sudo nmcli con add type ethernet ifname enp3s0 con-name mid360 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.50/24 \
  ipv4.never-default yes \
  autoconnect yes

sudo nmcli con up mid360
```

测试：

```bash
ip addr show enp3s0
ping 192.168.1.1XX
```

`192.168.1.1XX` 替换成雷达实际 IP。

## 29. Linux 快捷方式

Linux 中有两种常见“快捷方式”。

### 29.1 alias

命令快捷方式。适合简化进入容器：

```bash
alias denter='sudo docker start ros_cuda_env_user >/dev/null && sudo docker exec -it -u gugugaga -w /home/gugugaga ros_cuda_env_user bash'
```

### 29.2 软链接 ln -s

文件或目录快捷方式：

```bash
ln -s 真实路径 快捷方式路径
```

例子：

```bash
ln -s /home/gugugaga/catkin_ws ~/catkin_ws
```

删除软链接：

```bash
rm ~/catkin_ws
```

删除软链接不会删除原始目录。

## 30. 推荐阶段保存策略

因为你是手动搭环境，不要等全部装完才保存。推荐每完成一个大阶段就 commit 一次。

示例：

```text
nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04
  └─ mylab:base-user
       └─ mylab:ros-noetic
            └─ mylab:opencv47
                 └─ mylab:tensorrt861
                      └─ my_lab_k4a:latest
```

建议阶段：

```bash
sudo docker commit ros_cuda_env_user mylab:base-user
sudo docker commit ros_cuda_env_user mylab:ros-noetic
sudo docker commit ros_cuda_env_user mylab:opencv47
sudo docker commit ros_cuda_env_user mylab:tensorrt861
sudo docker commit ros_cuda_env_user my_lab_k4a:latest
```

如果后面装坏了，可以从上一个阶段镜像重新创建容器。

## 31. 当前推荐日常工作流

### 31.1 宿主机准备

新机器先装：

```text
Ubuntu 22.04 或 20.04
NVIDIA 驱动
Docker
NVIDIA Container Toolkit
SSH / RustDesk
必要 udev 规则
必要网卡 IP
```

验证 GPU 容器：

```bash
sudo docker run --rm --gpus all nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04 nvidia-smi
```

### 31.2 导入镜像

```bash
sudo docker load -i my_lab_k4a_latest.tar
```

### 31.3 创建容器

```bash
xhost +local:

sudo docker run -it --name ros_cuda_env_user \
  --gpus all \
  --network host \
  --ipc host \
  --privileged \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /dev:/dev \
  -v /dev/bus/usb:/dev/bus/usb \
  -v /run/udev:/run/udev:ro \
  -v /etc/localtime:/etc/localtime:ro \
  my_lab_k4a:latest \
  bash
```

### 31.4 日常进入

```bash
denter
```

等价于：

```bash
sudo docker start ros_cuda_env_user >/dev/null
sudo docker exec -it -u gugugaga -w /home/gugugaga ros_cuda_env_user bash
```

### 31.5 远程使用

另一台电脑：

```bash
ssh l@宿主机IP
denter
```

不要优先折腾容器内 SSH。

## 32. 常用命令速查

查看镜像：

```bash
docker images
```

查看容器：

```bash
docker ps
docker ps -a
```

启动容器：

```bash
sudo docker start ros_cuda_env_user
```

进入容器：

```bash
sudo docker exec -it -u gugugaga -w /home/gugugaga ros_cuda_env_user bash
```

停止容器：

```bash
sudo docker stop ros_cuda_env_user
```

删除容器：

```bash
sudo docker rm -f ros_cuda_env_user
```

保存容器为镜像：

```bash
sudo docker commit ros_cuda_env_user my_lab_k4a:latest
```

导出镜像：

```bash
sudo docker save -o my_lab_k4a_latest.tar my_lab_k4a:latest
```

导入镜像：

```bash
sudo docker load -i my_lab_k4a_latest.tar
```

查看 Docker 占用：

```bash
docker system df
```

查看镜像层：

```bash
docker history my_lab_k4a:latest
```

## 33. 一句话总结

你的路线是：

```text
宿主机只负责硬件和容器运行能力：
NVIDIA 驱动、Docker、NVIDIA Container Toolkit、udev、网络、SSH

容器负责复杂运行环境：
Ubuntu20.04、CUDA11.8、cuDNN、ROS、OpenCV、TensorRT、PCL、K4A、Livox、项目代码

手动在容器里装好并验证，清理缓存，然后 docker commit 成黄金镜像，再 docker save 拷到新机器。
```

这不是最优雅的 Dockerfile 自动化方案，但对你们这种历史依赖多、硬件多、安装步骤复杂的环境，是最务实的第一版。
