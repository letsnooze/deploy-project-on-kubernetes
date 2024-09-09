## 部署 RuoYi 至 Kubernetes 集群

欢迎使用这份全面的指南，它将指导您将 RuoYi 项目部署到 Kubernetes (k8s) 集群中。本仓库提供了一步步的流程，帮助您使用 Kubernetes 设置和管理您的部署。

### 项目概览

本项目旨在将 RuoYi 应用——一个流行的开源企业级快速开发平台——部署到 Kubernetes 集群中。按照提供的指南，您将能够有效地编排应用程序的容器。

### 前提条件

在开始之前，请确保您已经具备以下条件：

- 访问 Kubernetes 集群的权限。
- 安装了 Ansible 用于集群部署。
- 安装了 Docker 用于容器化。
- 拥有像 IntelliJ IDEA 这样的本地开发环境，用于创建 Docker 镜像。

### 部署流程

1. **使用 Ansible 部署集群**
   - 利用 `ansible` 目录下提供的 Ansible 剧本部署和配置您的 Kubernetes 集群。

2. **设置私有镜像仓库**
   - 在集群内的专用节点上安装并配置 Docker 镜像仓库。
   - 使用身份验证保护镜像仓库，确保只有授权访问。

3. **构建和推送 Docker 镜像**
   - 为 RuoYi 应用的后端和前端创建 Docker 镜像。
   - 将这些镜像推送到您的私有镜像仓库，供 Kubernetes 集群内使用。

4. **创建 Kubernetes 资源**
   - 应用 Kubernetes 清单文件创建必要的资源，如 ConfigMaps、Secrets 和 Deployments。
   - 配置服务以在集群内或外部公开您的应用程序。

### 详细步骤

1. **部署集群**

   - 执行 Ansible 剧本设置 Kubernetes 集群环境。

2. **安装 Docker 镜像仓库**

   ```bash
   apt-get update
   apt-get install apache2-utils
   ```

3. **配置镜像仓库身份验证**

   ```bash
   mkdir -p /docker/volume/registry/auth/
   htpasswd -Bc /docker/volume/registry/auth/htpasswd root
   ```

4. **设置镜像仓库数据和配置目录**

   ```bash
   mkdir -p /docker/volume/registry/data
   mkdir -p /docker/volume/registry/conf
   ```

5. **创建和配置 Docker 网络**

   ```bash
   docker network create registry-net
   ```

6. **启动镜像仓库容器**

   ```bash
   docker run -d \
     --name registry \
     --network registry-net \
     -v /docker/volume/registry/auth:/auth \
     -v /docker/volume/registry/data:/var/lib/registry \
     -v /docker/volume/registry/conf/config.yml:/etc/docker/registry/config.yml \
     -e REGISTRY_AUTH=htpasswd \
     -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
     -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
     -e REGISTRY_HTTP_SECRET=secretkey \
     -p 5000:5000 \
     registry:2
   ```

7. **配置 Docker 信任镜像仓库**

   ```bash
   vim /etc/docker/daemon.json
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "insecure-registries" : ["172.29.7.13:5000"]
   }
   ```

8. **安装并运行镜像仓库 UI**

   ```bash
   docker run -d \
     --name registry-browser \
     --network registry-net \
     -e REGISTRY_TITLE="Docker Registry Browser" \
     -e NGINX_PROXY_PASS_URL="http://registry:5000" \
     -e REGISTRY_AUTH="true" \
     -e REGISTRY_AUTH_USER="root" \
     -e REGISTRY_AUTH_PASSWORD="123456" \
     -p 8080:80 \
     joxit/docker-registry-ui:latest
   ```

9. **构建和推送 Docker 镜像**

   - 使用 IntelliJ IDEA 或其他兼容 Docker 的 IDE 构建后端和前端镜像。
   - 将镜像推送到您的私有镜像仓库。

10. **创建 Kubernetes 资源**

    - 应用 Kubernetes 清单文件设置集群内应用程序的环境。

    ```
    kubectl apply -f role-config.yaml
    kubectl create configmap ruoyi-init-sql-config-map --from-file=/opt/ruoyi/sql
    kubectl create configmap ruoyi-admin-config --from-file=/opt/ruoyi/application.yaml
    kubectl create configmap ruoyi-redis-config-map --from-file=/opt/ruoyi/redis.conf
    kubectl create secret docker-registry registry-user-pwd-secret \
      --docker-username=root \
      --docker-password=123456 \
      --docker-server=http://172.29.7.13:5000 
    kubectl apply -f ruoyi-k8s-backend.yaml
    kubectl create configmap ruoyi-ui-config --from-file=/opt/ruoyi/nginx.conf
    kubectl apply -f ruoyi-k8s-frontend.yaml
    ```

### 访问应用程序

部署完成后，通过访问集群任意工作节点上的暴露端口（例如，端口 30000）来使用应用程序。

### 常见问题与解决方案

1. **虚拟机内存不足**：如果登录页面提示系统接口问题，请确保您的虚拟机分配了足够的内存。如有必要，请调整内存设置并重启虚拟机。

2. **IDE 要求**：为了优化开发和部署，建议使用付费版的 IntelliJ IDEA，它提供了安全的 SSH 连接到 Docker 节点，以增强生产力。

按照这些指南，您应该能够成功地将 RuoYi 项目部署到您的 Kubernetes 集群。如果遇到任何问题，请参考提供的解决方案或寻求进一步的帮助。
