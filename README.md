# alicloud-go-demo 

## 一、项目概述

`alicloud-go-demo` 是一个简单的 Go 语言示例项目，本指南将详细介绍如何将该项目部署到阿里云服务器。整个部署流程主要借助 GitHub Actions 实现自动化构建、推送 Docker 镜像到阿里云容器镜像服务（ACR）个人版，再将镜像部署到阿里云弹性计算服务（ECS）实例。

## 二、前提条件

### 阿里云相关
- **阿里云账号**：需拥有一个已实名认证的阿里云账号。
- **ECS 实例**：创建一台 ECS 实例，安装好 Docker 环境。
- **ACR 个人版**：开通 ACR 个人版，创建命名空间，并获取访问凭证（用户名和密码）。

### GitHub 相关
- **GitHub 账号**：拥有一个 GitHub 账号，并且创建了包含 `alicloud-go-demo` 项目代码和 Dockerfile 的仓库。
- **GitHub Actions 权限**：确保你对该仓库有足够的权限来配置和运行 GitHub Actions。

## 三、配置步骤

### （一）配置阿里云

#### 1. ECS 实例配置
- 登录阿里云 ECS 控制台，创建或选择一台已有的 ECS 实例。
- 确保 ECS 实例的安全组规则允许访问 Docker 所需的端口（通常为 22 用于 SSH 连接，443 用于访问 ACR）。
- 在 ECS 实例上安装 Docker 环境，可参考阿里云官方文档。

#### 2. ACR 个人版配置
- 登录阿里云 ACR 控制台，开通 ACR 个人版。
- 创建一个命名空间，用于存放 `alicloud-go-demo` 项目的 Docker 镜像。
- 在 ACR 个人版的访问凭证管理页面，创建访问凭证，并记录用户名和密码。

### （二）配置 GitHub

#### 1. 生成 SSH 密钥对
- 在本地计算机上打开终端，执行以下命令生成 SSH 密钥对：

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

将生成的公钥添加到 ECS 实例的 ~/.ssh/authorized_keys 文件中，私钥用于后续配置 GitHub Secrets。

#### 2. 添加 GitHub Secrets

- 在 GitHub 仓库的 Settings -> Secrets -> Actions 中添加以下 Secrets：
    - ALICLOUD_ACR_REGISTRY：ACR 个人版的登录地址，如 registry.cn-hangzhou.aliyuncs.com。
    - ALICLOUD_ACR_NAMESPACE：在 ACR 个人版中创建的命名空间。
    - ALICLOUD_ACR_USERNAME：ACR 个人版的访问凭证用户名。
    - ALICLOUD_ACR_PASSWORD：ACR 个人版的访问凭证密码。
    - ECS_SSH_PRIVATE_KEY：ECS 实例的 SSH 私钥。
    - ECS_HOST：ECS 实例的公网 IP 地址。
    - ECS_USER：登录 ECS 实例的用户名，通常为 root。

#### 3. 创建 GitHub Actions 工作流文件
- 在项目的 .github/workflows 目录下创建 build-deploy.yml 文件，内容如下：
```yaml
name: Build, Push and Deploy Docker Image to Alibaba Cloud ACR Personal and ECS

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Alibaba Cloud ACR Personal
        uses: docker/login-action@v2
        with:
          registry: ${{ secrets.ALICLOUD_ACR_REGISTRY }}
          username: ${{ secrets.ALICLOUD_ACR_USERNAME }}
          password: ${{ secrets.ALICLOUD_ACR_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ secrets.ALICLOUD_ACR_REGISTRY }}/${{ secrets.ALICLOUD_ACR_NAMESPACE }}/alicloud-go-demo:${{ github.sha }}

  deploy-to-ecs:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Set up SSH key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.ECS_SSH_PRIVATE_KEY }}
          known_hosts: 'placeholder'

      - name: Add ECS host to known hosts
        run: ssh-keyscan -H ${{ secrets.ECS_HOST }} >> ~/.ssh/known_hosts

      - name: Login to Alibaba Cloud ACR Personal on ECS
        run: ssh ${{ secrets.ECS_USER }}@${{ secrets.ECS_HOST }} "docker login ${{ secrets.ALICLOUD_ACR_REGISTRY }} -u ${{ secrets.ALICLOUD_ACR_USERNAME }} -p ${{ secrets.ALICLOUD_ACR_PASSWORD }}"

      - name: Pull and run Docker image on ECS
        run: ssh ${{ secrets.ECS_USER }}@${{ secrets.ECS_HOST }} "docker pull ${{ secrets.ALICLOUD_ACR_REGISTRY }}/${{ secrets.ALICLOUD_ACR_NAMESPACE }}/alicloud-go-demo:${{ github.sha }} && docker stop alicloud-go-demo-container || true && docker rm alicloud-go-demo-container || true && docker run -d --name alicloud-go-demo-container -p 8080:8080 ${{ secrets.ALICLOUD_ACR_REGISTRY }}/${{ secrets.ALICLOUD_ACR_NAMESPACE }}/alicloud-go-demo:${{ github.sha }}"

```

## 四、部署流程
1. 将代码推送到 GitHub 仓库的 main 分支，GitHub Actions 会自动触发工作流。
2. 构建和推送镜像：工作流的 build-and-push 任务会拉取代码，设置 Docker Buildx，登录到 ACR 个人版，然后构建并推送 Docker 镜像到 ACR 个人版。
3. 部署到 ECS：deploy-to-ecs 任务会在镜像构建和推送成功后执行，它会设置 SSH 密钥，登录 ECS 实例，在 ECS 实例上登录 ACR 个人版，拉取最新的 Docker 镜像，停止并删除旧的容器，最后启动新的容器。