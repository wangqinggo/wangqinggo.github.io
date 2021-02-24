[..](./../middleware/index.md)
## Docker

### 实现机制

- namespaces

- CGroups

- AUFS

### 基本操作

- 拉取镜像

- 打包镜像

- 运行镜像

- 发布镜像

### Dockerfile

- FROM

- ARG | ENV（作用域不同）

- WORKDIR

- RUN

- COPY | ADD（优先COPY）

- VOLUME

- EXPOSE

- ENTRYPOINT | CMD(ENTRYPOINT优先)

### 实践

- 保持容器简单

- 固定软件版本

- 整理Dockerfile语句

- 使用多阶段构建 

### 网络模式

#### 独立的Network Namespace

- Bridge模式

  - docker 的默认网络模式

  - 创建docker0的虚拟网桥

- Container 模式 : 多个容器共享Network Namespace

- None模式 : 配置为空，需要自己操作

#### Host 模式

> 使用宿主机的 IP 和端口