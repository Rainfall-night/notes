# 24 Web 部署

> **系列**：web与后台服务 专题（ASP.NET Core 专属）｜ **上一篇**：[[23-认证授权]]

### 背景

> 后台服务（Worker）的部署见 [[15-部署]]（Windows Service / Systemd）。本篇讲 **ASP.NET Core Web 应用的部署**：发布模式、Kestrel、IIS、Nginx、Docker、K8s、环境配置——Web 有 HTTP 入口，部署思路和后台服务差异很大。

### 1. 发布模式

| 模式 | 说明 | 特点 |
|---|---|---|
| **Framework-dependent** | 依赖目标机装 .NET 运行时 | 体积小、更新运行时麻烦 |
| **Self-contained** | 自带运行时 | 体积大、目标机无需装 .NET |
| 单文件 | 全部打成一个 exe | 便于分发，启动慢一点 |

```bash
dotnet publish -c Release -o ./publish          # 默认框架依赖
dotnet publish -c Release -r win-x64 --self-contained   # 自包含
```

### 2. Kestrel：内置 Web 服务器

- ASP.NET Core 内置的高性能跨平台 Web 服务器（Kestrel）
- 端口在 `appsettings.json` 配置：

```json
{
  "Kestrel": {
    "Endpoints": {
      "Http": { "Url": "http://*:8080" },
      "Https": { "Url": "https://*:8081" }
    }
  }
}
```

- ⚠️ 生产环境**一般不直接用 Kestrel 对公网**：HTTPS 证书、静态文件、请求缓冲等交给专业服务器（IIS / Nginx）做反代

### 3. IIS 部署（Windows）

- Windows 上通过 **IIS** 托管 ASP.NET Core
- 原理：**ASP.NET Core Module（ANCM）** 把请求转发给 Kestrel

```
IIS ──> ANCM ──> Kestrel
```

- 两种托管模式：
  - **进程内（in-process）**：Kestrel 跑在 IIS 工作进程里，性能好（默认）
  - **进程外（out-of-process）**：独立 Kestrel 进程，ANCM 反代
- 部署产物里要包含 `web.config`（ANCM 配置）

### 4. 反向代理：Nginx

- Linux 上常用 **Nginx 反向代理**到 Kestrel

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://127.0.0.1:5000;   # 转发到 Kestrel
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

- 由 Nginx 处理公网流量、静态文件、HTTPS，Kestrel 专注业务

### 5. Docker 容器化

多阶段构建：先用 sdk 镜像编译，再用 aspnet 镜像运行（体积小）：

```dockerfile
# 阶段 1：构建
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

# 阶段 2：运行
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyWebApp.dll"]
```

### 6. Kubernetes 部署

- **Deployment**（副本数）、**Service**（负载均衡）、**Ingress**（外部入口）
- **探针**：`livenessProbe`（活了没）、`readinessProbe`（能接流量了没）——这就是 [[05-异常处理]] 看门狗的容器版
- **水平伸缩**（HPA）：按 CPU/请求数自动加副本

```yaml
livenessProbe:
  httpGet: { path: /health, port: 8080 }   # 健康检查端点
```

### 7. 环境与配置

- 环境变量 `ASPNETCORE_ENVIRONMENT` / `DOTNET_ENVIRONMENT` 切换环境
- `appsettings.{Environment}.json` 按环境覆盖配置
- 敏感信息（密码、连接串）用**环境变量 / K8s Secret**，不写进配置文件

### 8. Web vs 后台服务部署对比

| 维度 | Web | 后台服务 |
|---|---|---|
| 入口 | HTTP 端口（Kestrel） | 无端口，系统管理 |
| Windows | IIS / ANCM | Windows Service |
| Linux | Nginx + systemd | systemd |
| 容器 | 必须暴露端口 | 常驻进程，无端口 |
| 探活 | liveness/readiness 探针 | 自写看门狗/心跳 |

### 9. 小结

| 知识点 | 一句话 |
|---|---|
| 发布模式 | 框架依赖 vs 自包含 vs 单文件 |
| Kestrel | 内置服务器，生产用反代包一层 |
| IIS | ANCM 反代到 Kestrel（in-process） |
| Nginx | Linux 反代，管 HTTPS/静态文件 |
| Docker | 多阶段构建，aspnet 运行镜像 |
| K8s | 探针 + 水平伸缩 |
| 环境 | ASPNETCORE_ENVIRONMENT + Secret |
