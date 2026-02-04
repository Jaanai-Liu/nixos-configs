# 🚀 NixOS Deployment Guide for lz-pc

本指南用于在全新的 NixOS 环境中（如 VMware 虚拟机）快速引导并部署 `lz-pc` 配置。

## 前置准备 (Pre-flight Check)
* **硬盘空间**：虚拟机硬盘建议 **>100GB**（构建 NixOS 需要大量临时空间，64GB 可能会导致构建失败）。
* **网络环境**：建议宿主机开启代理，或确保能正常访问 GitHub。

---

## 第一阶段：环境自举 (Bootstrap)

在全新安装的系统中，我们需要先手动开启 Flakes 功能、安装 Git/Vim，并配置清华大学镜像源以加速下载。

### 1. 编辑基础配置
使用系统自带的编辑器修改配置：

```bash
sudo nano /etc/nixos/configuration.nix

```

### 2. 写入引导配置

在配置文件中找到适当位置（如 `environment.systemPackages` 附近），添加或修改以下内容。这将启用 Flakes、安装基础工具并配置国内加速缓存。

```nix
{ config, pkgs, ... }:

{
  # ... 原有配置 ...

  # 1. 开启 Flakes 和新版 Nix 命令行工具
  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  # 2. 安装基础软件
  environment.systemPackages = with pkgs; [
    git
    vim
    wget
    curl
  ];

  # 3. 配置清华大学 Binary Cache (极大加速软件下载)
  nix.settings.substituters = [ "[https://mirrors.tuna.tsinghua.edu.cn/nix-channels/store](https://mirrors.tuna.tsinghua.edu.cn/nix-channels/store)" ];
  nix.settings.trusted-public-keys = [ "mirrors.tuna.tsinghua.edu.cn-1:ly95/J/qf50BIncRiK6i13Wz+89I4VjGEN0S8sA+vJc=" ];

  # ... 原有配置 ...
}

```

> **注意**：编辑完成后，按 `Ctrl+O` 保存，`Ctrl+X` 退出。

### 3. 应用更改

运行旧版构建命令使上述配置生效：

```bash
sudo nixos-rebuild switch

```

---

## 第二阶段：配置 GitHub 访问

为了拉取您的私有配置或推送代码，需要配置 SSH 密钥。

### 1. 设置 Git 身份

```bash
git config --global user.name "YourGithubName"
git config --global user.email "your@email.com"

```

### 2. 生成 SSH 密钥

```bash
# 一路回车即可，无需设置密码
ssh-keygen -t ed25519 -C "your@email.com"

```

### 3. 添加密钥到 GitHub

读取公钥内容：

```bash
cat ~/.ssh/id_ed25519.pub

```

* 复制输出的内容（以 `ssh-ed25519` 开头）。
* 在浏览器打开 [GitHub SSH Keys 设置页面](https://github.com/settings/ssh)。
* 点击 **New SSH key**，粘贴内容并保存。

### 4. 测试连接

```bash
ssh -T git@github.com
# 看到 "Hi xxx! You've successfully authenticated..." 即表示成功

```

---

## 第三阶段：部署系统 (Deploy)

### 1. 拉取配置仓库

```bash
cd ~
git clone git@github.com:YourGithubName/nixos-configs.git
cd nixos-configs

```

### 2. 检查主机配置

确保 `hosts/lz-pc/` 目录存在，且 `outputs/x86_64-linux/hosts/lz-pc.nix` 配置正确。

### 3. 构建并切换系统

这是最关键的一步。系统将根据 Flake 配置下载所有依赖并重构系统。

* `--flake .#lz-pc`: 使用当前目录下的 flake 文件，构建名为 `lz-pc` 的主机。
* `--show-trace`: 如果出错，显示详细的错误堆栈。

```bash
sudo nixos-rebuild switch --flake .#lz-pc --show-trace

```

> **提示**：第一次构建时间较长，请耐心等待。如果因为网络原因下载 GitHub 源码失败，请检查网络代理。

---

## 日常维护与更新 (Maintenance)

部署完成后，日常使用以下命令进行维护。

### 仅修改配置文件 (不升级软件版本)

当您修改了 `home/my-apps.nix` 添加新软件，或修改了桌面配置时：

```bash
sudo nixos-rebuild switch --flake .#lz-pc

```

### 彻底系统更新 (升级内核与软件版本)

当您想要获取 NixOS 的最新软件版本时（这将更新 `flake.lock` 文件）：

```bash
# 1. 更新锁文件 (Update flake.lock)
# 由于配置了清华源，后续构建会走国内镜像，但此步仍需访问 GitHub 获取版本号
nix flake update

# 2. 构建新系统
sudo nixos-rebuild switch --flake .#lz-pc

```
