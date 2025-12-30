# 🔐Renew Certs Actions

**自动化证书续期**

---

## 🎯 项目概览

本项目针对 **Cloudflare 持有的域名** 提供全自动化的 Let's Encrypt 证书续期方案，无需手动干预，支持多种证书分发方式。

### 两种工作流模式

| 工作流                          | 说明                                 | 适用场景                                   |
| ------------------------------- | ------------------------------------ | ------------------------------------------ |
| **renew_cert.yml**        | 默认工作流，与 AetherVault SaaS 集成 | 使用 AetherVault 进行统一证书分发与管理    |
| **renew_cert_custom.yml** | 自定义存储入口（需自行实现上传逻辑） | 集成私有存储服务（S3、MinIO、自建 API 等） |

### 选择您的工作流

- 使用 **AetherVault** 作为证书中心？→ 部署 `renew_cert.yml`
- 希望集成自定义存储服务？→ 参考 `renew_cert_custom.yml` 修改上传逻辑

---

## 📖 设计理念与目标

Renew Certs Actions + AetherVault 是专为 **个人开发者** 和 **小规模技术团体** 设计的自动化运维方案。

### 核心目标

- **精力回收**：通过 GitHub Actions 自动处理 Let's Encrypt 证书申请与续期，彻底告别手动更新，防止各种因忘记续费/断电/死机等常见问题导致续期中断。
- **跨网分发**：证书生成后自动同步至基于 Cloudflare Workers 的 AetherVault SaaS（或自定义存储）。无论您的服务器是在家庭内网（如 NAS）还是公网多台小型 VPS，均可统一拉取。
- **安全隔离**：通过"一机一码（UUID）"和"私有仓库"模式，将自动化权限与核心资产严格隔离。

---

### ⚠️ 免责声明：非企业级解决方案

**本项目基于个人开发者的"低维护成本"需求构建。不建议在企业生产环境或对合规性有极高要求的场景下使用：**

- 本方案假设节点间通过 UUID 向Vault进行简单鉴权，未引入复杂的 HSM（硬件安全模块）或 CA 层级等设计。
- 不提供 SLA 保证，安全性依赖于 Cloudflare 基础设施及个人对 Token 的保管。
- 不支持企业级审计日志、PKI 体系或合规性认证。

---

## 🛠️ 快速开始步骤

为了确保您的凭据安全，**本项目强制要求在私有仓库中运行。**

### 第一步：克隆并私有化仓库

**源仓库地址：**

```
https://github.com/wuyilingwei/AUTO_RENEW_CERTS
```

**导入步骤：**

1. 复制上方仓库地址
2. 访问 GitHub 导入页面：https://github.com/new/import
3. 在 "Repository URL" 字段粘贴上方地址
4. 输入您的新仓库名称（例如 `my-cert-auto-renew`）
5. **重要：选择 Private (私有) 仓库** ⚠️
6. 点击 "Begin Import" 完成导入

### 第二步：配置 GitHub Secrets & Variables

进入您的私有仓库 **Settings > Secrets and variables > Actions**，配置以下信息：

#### 2.1 Variables (环境变量)

##### `VAULT_URL`

- **说明**：您的 AetherVault 服务地址
- **示例**：`https://vault.your-domain.com`
- **作用**：接收证书数据的后端服务地址

##### `DOMAINS_CONFIG`

- **说明**：证书申请的 JSON 配置，定义需要申请的域名及其对应的 Cloudflare API Token
- **格式**：JSON 数组
- **示例**：

```json
[
  {
    "domain": "example.com",
    "email": "dev@example.com",
    "cf_token": "YOUR_CLOUDFLARE_DNS_TOKEN",
    "apply_domains": ["example.com", "*.example.com"]
  },
  {
    "domain": "another.com",
    "email": "admin@another.com",
    "cf_token": "ANOTHER_CF_TOKEN",
    "apply_domains": ["another.com", "www.another.com", "api.another.com"]
  }
]
```

**字段说明**：

- `domain`：主域名标识符（用于本地证书目录命名及 Vault 中的 key 前缀）
- `email`：Let's Encrypt 注册邮箱（用于证书通知）
- `cf_token`：Cloudflare API Token（详见下方权限范围说明）
- `apply_domains`：申请证书的完整域名列表（包含主域名和所有二级域名）

##### `VAULT_TOKEN`

- **说明**：从 AetherVault 后端生成的具有 **w (写入)** 权限，范围为 `/certs/*`的 UUID Token
- **用途**：用于 HTTP Bearer 认证，允许向 Vault 上传证书数据
- **安全建议**：
  - 在 AetherVault 后端为 Actions 分配专用 UUID，勿与其他操作共用同一 Token
  - 仅为该 Token 赋予 `/certs/*` 路径的写入权限

> **⚠️ 仅使用 AetherVault 时需配置** `VAULT_TOKEN` 和 `VAULT_URL`
> 如使用自定义存储工作流（`renew_cert_custom.yml`），此步骤可跳过。

### 第三步：选择并激活工作流

进入仓库的 **Actions** 页面，根据您的需求选择工作流：

#### 选项 A：使用 AetherVault（推荐新手）

如果您已配置好 AetherVault 服务和相关 Secrets：

1. 选择 **"Cert Renewal to Custom Vault"** 工作流
2. 点击 **"Run workflow"** 手动触发第一次申请
3. 监控日志确保证书申请成功，您可以在AetherVault的Database面板查看结果。
4. 工作流将根据 `cron: '0 0 1 * *'` 每月自动运行

#### 选项 B：使用自定义存储（高级用户）

如果您希望集成自定义存储服务（S3、MinIO、自建 API 等）：

1. 编辑 [.github/workflows/renew_cert_custom.yml](.github/workflows/renew_cert_custom.yml)
2. 在 "3. 上传至自定义存储服务" 部分实现您的上传逻辑
3. 证书文件位置：
   ```
   ./certs/live/{domain}/
   ├── privkey.pem      # 私钥
   ├── cert.pem         # 证书
   └── fullchain.pem    # 完整证书链
   ```
4. 参考文件中的示例和注释完成实现
5. 点击 **"Run workflow"** 手动测试
6. 确认成功后，工作流将自动定时运行

---

## 🔑 Cloudflare API Token 权限范围（重要）

### 权限最小化原则

为了防止 Token 泄露导致的域名劫持或恶意记录修改，**强烈建议为 Certbot 创建单独的 Cloudflare API Token**，并仅授予以下最小权限：

### 必需权限

| 权限项                       | 范围             | 目的                               |
| ---------------------------- | ---------------- | ---------------------------------- |
| **Zone > DNS > Edit**  | 特定区域（Zone） | 允许修改 DNS TXT 记录（ACME 验证） |
| **Zone > Zone > Read** | 特定区域（Zone） | 允许读取区域信息（Certbot 初始化） |

### Cloudflare Token 创建步骤

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
2. 进入 **Account Settings > API Tokens**
3. 点击 **Create Token**，选择 **Custom token**
4. 配置权限：

   ```
   Permissions:
   - Zone > DNS > Edit
   - Zone > Zone > Read

   Zone Resources:
   - Include > Specific zone > [选择需要续期的域名区域]
   ```
5. 将生成的 Token 添加到 `DOMAINS_CONFIG` 的 `cf_token` 字段

---

## 📊 执行流程

```
┌─────────────────┐
│  触发工作流     │  手动触发或定时触发
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  1. 读取 DOMAINS_CONFIG 配置        │
│     解析域名列表及对应的 CF Token   │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  2. 遍历每个域名配置                │
│     调用 Certbot + Cloudflare       │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  3. Certbot 申请证书                │
│     - DNS TXT 验证（ACME Challenge）│
│     - 获取 privkey.pem, cert.pem    │
│     - 保存至 ./certs/live/{domain}  │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  4a. AetherVault 上传               │  renew_cert.yml
│     - 构造 JSON payload             │
│     - 使用 VAULT_TOKEN 认证         │
│     - POST 至 /api/data            │
└────────┬────────────────────────────┘
         │                    │
         │                    └──────────┬────────────────┐
         │                               │                │
         │                               ▼                ▼
         │                    ┌──────────────────────────────────┐
         │                    │ 4b. 自定义存储上传               │
         │                    │     renew_cert_custom.yml         │
         │                    │   - 调用自定义上传脚本            │
         │                    │   - 支持 S3, MinIO, API 等       │
         │                    └──────────────────────────────────┘
         │                               │
         └───────────────────────────────┘
                        │
                        ▼
         ┌─────────────────────────────────────┐
         │  5. 生成执行报告                    │
         │     列出成功/失败的域名及状态       │
         └─────────────────────────────────────┘
```

---

## � 自定义集成指南

如果您不使用 AetherVault，可以参考本章节实现自定义的证书分发方式。

### 修改 renew_cert_custom.yml

1. 编辑 `.github/workflows/renew_cert_custom.yml`
2. 找到"**3. 上传至自定义存储服务（TODO: 修改此部分）**"注释
3. 在该部分添加您的上传逻辑，证书内容已在以下变量中：

   - `$PRIVKEY` : 私钥 PEM 内容
   - `$CERT` : 证书 PEM 内容
   - `$FULLCHAIN` : 完整证书链 PEM 内容
4. 添加必要的 GitHub Secrets（如 AWS Access Key、自定义 API Token 等）
5. 在工作流中使用 Secrets（例如 `${{ secrets.AWS_ACCESS_KEY }}`）
6. 测试工作流，确保证书成功上传
