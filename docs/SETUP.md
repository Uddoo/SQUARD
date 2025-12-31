# Powder Toy 开发环境设置指南

更多项目文档索引见：`docs/README.md`。

## 前置要求

### 1. 安装 direnv
```bash
# Ubuntu/Debian
sudo apt install direnv

# 添加到 shell 配置
# 对于 bash，添加到 ~/.bashrc
# 对于 zsh，添加到 ~/.zshrc
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc  # 如果使用 zsh
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc  # 如果使用 bash

# 重新加载 shell 配置
source ~/.zshrc  # 或 source ~/.bashrc
```

### 2. 安装系统依赖

运行以下命令安装所有必需的系统依赖：

```bash
sudo apt update
sudo apt install -y \
    g++ \
    git \
    python3 \
    python3-pip \
    python3-venv \
    pkg-config \
    ccache \
    libluajit-5.1-dev \
    libcurl4-openssl-dev \
    libssl-dev \
    libfftw3-dev \
    libsdl2-dev \
    libbz2-dev \
    libjsoncpp-dev \
    libpng-dev
```

## 激活开发环境

### 首次使用

1. 进入项目目录：
```bash
cd /home/uddo/Code/Dev/SQUARD
```

2. 允许 direnv 加载配置：
```bash
direnv allow
```

这将自动：
- 创建 Python 虚拟环境 (`.venv`)
- 安装 Python 依赖（meson, ninja）
- 配置环境变量

### 日常使用

每次进入项目目录时，direnv 会自动激活开发环境。离开目录时自动停用。

## 构建项目

```bash
# 配置构建
meson setup build

# 编译
meson compile -C build

# 运行
./build/powder
```

## 可选配置

### 使用 ccache 加速编译

ccache 已在 .envrc 中自动配置。查看缓存统计：
```bash
ccache -s
```

### 清理构建

```bash
rm -rf build .ccache
```

## 故障排除

### direnv 未生效
```bash
# 检查 direnv 是否正确安装
which direnv

# 检查 shell hook 是否配置
direnv status
```

### Python 依赖安装失败
```bash
# 手动重新安装
.venv/bin/pip install -r requirements.txt
```

### 系统依赖缺失
检查所有依赖是否已安装：
```bash
dpkg -l | grep -E "g\+\+|libluajit|libcurl|libssl|libfftw3|libsdl2|libbz2|libjsoncpp|libpng"
```
