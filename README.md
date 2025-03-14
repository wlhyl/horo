# Horo - 占星学应用套件

本项目是一个基于微服务架构的占星学应用套件，包含多个子模块，用于提供占星盘计算、绘制和案例管理功能。

## 项目结构

本项目由以下子模块组成：

### horo-api

基于Rust语言和Actix Web框架开发的占星API服务，提供以下功能：

- 本命盘绘制
- 推运盘绘制（法达、小限、日返、月返、比较盘）
- 七政四余盘星盘绘制

### horo-ui

基于Ionic/Angular开发的占星应用前端，提供：

- 本命盘与各类推运盘的绘制界面
- 七政四余盘星盘绘制
- 行星必然尊贵表
- 案例库界面

### horo-storage-api

案例存储服务的后端API，为horo-ui提供案例存储功能，使用MySQL数据库。

### horo-storage-ui

案例存储服务的管理前端，为horo-storage-api提供Web界面。

### horo-chart

Helm Chart部署配置，用于在Kubernetes环境中部署整个应用套件。

## 环境准备

在开始部署前，需要准备以下环境：

1. 下载并编译瑞士星历表
```bash
mkdir /path/to/swe
cd /path/to/swe
wget https://github.com/aloistr/swisseph/archive/refs/tags/v2.10.03.tar.gz -O swe.tar.gz
tar xvzf swe.tar.gz
cd swisseph-2.10.03
make libswe.a
```

2. 下载星历表文件
```bash
cd /path/to/swe
wget https://raw.githubusercontent.com/aloistr/swisseph/master/ephe/ephe/semo_18.se1
wget https://raw.githubusercontent.com/aloistr/swisseph/master/ephe/ephe/semom48.se1
wget https://raw.githubusercontent.com/aloistr/swisseph/master/ephe/ephe/sepl_18.se1
wget https://raw.githubusercontent.com/aloistr/swisseph/master/ephe/ephe/seplm48.se1
```

3. 将计算恒星的文件复制到星历表目录
```bash
wget https://raw.githubusercontent.com/wlhyl/horo-api/main/sefstars.txt -O /path/to/swe/sefstars.txt
```

4. 准备MySQL数据库（用于horo-storage-api）

## 完整部署步骤

### 1. 克隆仓库并更新子模块

```bash
# 克隆主仓库
git clone https://github.com/your-username/horo.git
cd horo

# 初始化子模块
git submodule init

# 更新所有子模块到最新版本
git submodule update --remote
```

### 2. 部署方式选择

#### 方式一：本地开发部署

1. 部署horo-api服务

```bash
cd horo-api

# 创建日志配置文件
cat > log4rs.yaml << EOF
---
refresh_rate: 30 seconds
appenders:
  stdout:
    kind: console
root:
  level: info
  appenders:
    - stdout
EOF

# 运行API服务
LOG4RS_CONFIG=$(pwd)/log4rs.yaml \
  EPHE_PATH=/path/to/swe \
  RUSTFLAGS=-L/path/to/swe/src \
  cargo run --features swagger,cors
```

2. 部署horo-storage-api服务

```bash
cd ../horo-storage-api

# 创建日志配置文件
cat > log4rs.yaml << EOF
---
refresh_rate: 30 seconds
appenders:
  stdout:
    kind: console
root:
  level: info
  appenders:
    - stdout
EOF

# 迁移数据库
DATABASE_URL="mysql://db_user:db_password@db_host/db_name" \
  cargo run --bin migration

# 运行存储API服务
LOG4RS_CONFIG=$(pwd)/log4rs.yaml \
  DATABASE_URL="mysql://db_user:db_password@db_host/db_name" \
  TOKEN_EXPIRE_SECONDS=86400 \
  USERNAME="your_username" \
  PASSWORD="your_password" \
  cargo run --features swagger,cors \
  --bin storage_api -- -p 8081
```

3. 部署horo-ui前端

```bash
cd ../horo-ui

# 安装依赖
npm install

# 启动服务
ionic serve
```

4. 部署horo-storage-ui前端（可选）

```bash
cd ../horo-storage-ui

# 安装依赖
npm install

# 启动服务
ng serve --open
```

#### 方式二：Kubernetes部署

使用项目提供的Helm Chart在Kubernetes集群中部署：

```bash
# 进入chart目录
cd horo-chart

# 安装chart
helm install horo . \
  --namespace horo \
  --create-namespace \
  --set database.host=mysql \
  --set database.database=horo \
  --set database.user=root \
  --set database.password=password \
  --set storage.user=admin \
  --set storage.password=admin \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set ingress.host=your-hostname.com
```

查看horo-chart/README.md获取更详细的配置选项。

## 项目更新

每次部署项目时，都应该将各子模块更新到最新版本：

```bash
# 在主仓库目录下执行
git pull

# 更新所有子模块到最新版本
git submodule update --remote

# 如果有修改，可以提交更新
git add .
git commit -m "Update all submodules to latest version"
git push
```

## 许可证

项目使用GPL-3.0 许可证。
