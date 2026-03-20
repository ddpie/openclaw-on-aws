# OpenClaw on AWS

> 在 AWS 上部署 [OpenClaw](https://github.com/openclaw/openclaw)，使用 Amazon Bedrock。无需 API Key，IAM 认证。

[OpenClaw](https://docs.openclaw.ai) 是一个开源 AI 助手平台，支持连接多种大语言模型，并通过 WhatsApp、Telegram、Discord、飞书等渠道与用户交互。本仓库提供 CloudFormation 模板，帮助你在 AWS 上快速部署 OpenClaw。

[中文](#中文) | [English](#english)

[![OpenClaw](https://img.shields.io/badge/OpenClaw-AI_Assistant-purple.svg)](https://github.com/openclaw/openclaw)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![AWS](https://img.shields.io/badge/AWS-Bedrock-orange.svg)](https://aws.amazon.com/bedrock/)
[![CloudFormation](https://img.shields.io/badge/IaC-CloudFormation-blue.svg)](https://aws.amazon.com/cloudformation/)

---

<a id="中文"></a>

## 部署方案

| | 独立部署 | 共享 VPC（多人） |
|---|---|---|
| **适用场景** | 单人使用 | 团队（10–100+ 人） |
| **模板** | `openclaw-standalone.yaml` | `vpc-shared.yaml` + `openclaw-per-user.yaml` |
| **VPC** | 每人独立 | 全员共享 |
| **隔离性** | 完全网络隔离 | IAM + 数据卷隔离，网络共享 |

## 前置条件

1. 在 [Bedrock 控制台 - Model Access](https://console.aws.amazon.com/bedrock/home#/modelaccess) **开启模型访问**（默认使用 Amazon Nova 2 Lite，追求最佳效果可选 Claude Opus 4.6）
2. 在目标 Region **创建 EC2 密钥对**（可选，用于 SSH 访问）

## 方案一：独立部署（单人）

一个 Stack = 一套完整的 OpenClaw（含独立 VPC）。适合个人使用或快速体验。

### 控制台部署

1. 打开对应 Region 的 CloudFormation 控制台：
   - [美西 Oregon](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create)
   - [美东 Virginia](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create)
   - [欧洲 Ireland](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create)
   - [亚太 Tokyo](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create)
2. 选择 **Upload a template file**，上传 `openclaw-standalone.yaml`
3. Stack 名称：`openclaw-bedrock`
4. 配置参数（模型、实例类型等）
5. 勾选 **"I acknowledge that AWS CloudFormation might create IAM resources"**
6. 点击 **Create stack**，等待约 8 分钟至 `CREATE_COMPLETE`

### CLI 部署

```bash
git clone https://github.com/ddpie/openclaw-on-aws.git
cd openclaw-on-aws

aws cloudformation create-stack \
  --stack-name openclaw-bedrock \
  --template-body file://openclaw-standalone.yaml \
  --parameters \
    ParameterKey=InstanceType,ParameterValue=t4g.medium \
  --capabilities CAPABILITY_IAM
```

### 部署后操作

> 💡 **前置条件**：需安装 [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) 和 [Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

```bash
# 1. 获取实例 ID
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-bedrock \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)

# 2. 端口转发（在本地终端运行，保持窗口打开）
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'

# 3. 获取 access token（新开一个终端）
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartInteractiveCommand \
  --parameters command='["sudo -u ubuntu cat /home/ubuntu/.openclaw/openclaw.json | grep token"]'

# 4. 打开浏览器访问 http://localhost:18789，输入 token 登录

# 或者直接 SSM 登录实例进行初始化
aws ssm start-session --target $INSTANCE_ID
sudo su - ubuntu
openclaw onboard
```

## 方案二：共享 VPC（多人部署）

一套共享网络 + N 个用户实例。每人独立 EC2、IAM Role、EBS 数据卷。

### 第一步 — 部署共享网络（仅一次）

1. 打开对应 Region 的 CloudFormation 控制台：
   - [美西 Oregon](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create)
   - [美东 Virginia](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create)
   - [欧洲 Ireland](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create)
   - [亚太 Tokyo](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create)
2. 上传 `vpc-shared.yaml`，Stack 名称：`openclaw-shared-vpc`
3. 等待 `CREATE_COMPLETE`（约 2 分钟）
4. 进入 **Outputs** 页签，记录：`VpcId`、`PublicSubnet1Id`、`PublicSubnet1AZ`

### 第二步 — 部署用户实例

1. 创建新 Stack，上传 `openclaw-per-user.yaml`
2. Stack 名称：`openclaw-用户名`（如 `openclaw-alice`）
3. 填写参数：
   - `VpcId` — 第一步 Outputs 中的值
   - `SubnetId` — 第一步 Outputs 中的 `PublicSubnet1Id`
   - `AvailabilityZone` — 第一步 Outputs 中的 `PublicSubnet1AZ`
   - `InstanceType` — `t4g.medium`（最低要求）
4. 勾选 **"I acknowledge that AWS CloudFormation might create IAM resources"**
5. 等待 `CREATE_COMPLETE`（约 8 分钟）
6. 每个用户重复以上步骤

### 批量部署脚本

```bash
git clone https://github.com/ddpie/openclaw-on-aws.git
cd openclaw-on-aws

# 第一步：共享网络
aws cloudformation create-stack \
  --stack-name openclaw-shared-vpc \
  --template-body file://vpc-shared.yaml

# 等待完成后获取输出
VPC_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-shared-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)
SUBNET_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-shared-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`PublicSubnet1Id`].OutputValue' --output text)
AZ=$(aws cloudformation describe-stacks \
  --stack-name openclaw-shared-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`PublicSubnet1AZ`].OutputValue' --output text)

# 第二步：批量创建用户实例
for USER in alice bob charlie; do
  aws cloudformation create-stack \
    --stack-name openclaw-$USER \
    --template-body file://openclaw-per-user.yaml \
    --parameters \
      ParameterKey=VpcId,ParameterValue=$VPC_ID \
      ParameterKey=SubnetId,ParameterValue=$SUBNET_ID \
      ParameterKey=AvailabilityZone,ParameterValue=$AZ \
      ParameterKey=InstanceType,ParameterValue=t4g.medium \
    --capabilities CAPABILITY_IAM
done
```

### 第三步 — 初始化每个实例

```bash
# 获取实例 ID
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-alice \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' --output text)

# 端口转发（本地终端运行，保持窗口打开）
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'

# 获取 access token（新开终端）
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartInteractiveCommand \
  --parameters command='["sudo -u ubuntu cat /home/ubuntu/.openclaw/openclaw.json | grep token"]'

# 打开 http://localhost:18789 输入 token 登录

# 或直接 SSM 登录初始化
aws ssm start-session --target $INSTANCE_ID
sudo su - ubuntu
openclaw onboard
```

### 删除顺序

> ⚠️ 先删**所有用户 Stack**，最后删共享 VPC Stack。

## 注意事项

| 项目 | 说明 |
|------|------|
| **最低实例规格** | `t4g.medium`（4GB 内存）。`t4g.small`（2GB）会导致 Gateway OOM。 |
| **子网容量** | 每个 /24 子网约 251 个实例，超出需添加子网。 |
| **VPC Endpoint** | 关闭时 Bedrock/SSM 走公网，需确保有公网 IP。生产环境建议开启。 |
| **删除保护** | 建议对共享 VPC Stack 开启 Termination Protection。 |
| **部署后操作** | SSM 登录 → `sudo su - ubuntu` → `openclaw onboard` |

## 架构

```
┌─────────────────────────────────────────────────┐
│                   AWS Account                    │
│  ┌────────────────────────────────────────────┐  │
│  │            VPC（共享或独立）                 │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │  │
│  │  │  用户 A  │  │  用户 B  │  │  用户 C  │ │  │
│  │  │ EC2+IAM  │  │ EC2+IAM  │  │ EC2+IAM  │ │  │
│  │  │  +EBS    │  │  +EBS    │  │  +EBS    │ │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘ │  │
│  │       │              │              │       │  │
│  │  ┌────┴──────────────┴──────────────┴────┐  │  │
│  │  │       VPC Endpoints（可选）            │  │  │
│  │  │   Bedrock · SSM · SSMMessages · S3     │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────┘  │
│                       │                          │
│              ┌────────┴────────┐                 │
│              │  Amazon Bedrock │                 │
│              │  Claude · Nova  │                 │
│              └─────────────────┘                 │
└─────────────────────────────────────────────────┘
```

---

<a id="english"></a>

# OpenClaw on AWS (English)

> Deploy [OpenClaw](https://github.com/openclaw/openclaw) on AWS with Amazon Bedrock. No API keys. IAM authentication.

[OpenClaw](https://docs.openclaw.ai) is an open-source AI assistant platform that connects to multiple LLMs and interacts with users via WhatsApp, Telegram, Discord, Lark, and more. This repo provides CloudFormation templates for quick deployment on AWS.

## Deployment Options

| | Standalone | Shared VPC (Multi-User) |
|---|---|---|
| **Use case** | Single user | Teams (10–100+ users) |
| **Template** | `openclaw-standalone.yaml` | `vpc-shared.yaml` + `openclaw-per-user.yaml` |
| **VPC** | One per stack | Shared across all users |
| **Isolation** | Full network isolation | IAM + EBS isolation, shared network |

## Prerequisites

1. **Enable Bedrock models** in the [Bedrock Console - Model Access](https://console.aws.amazon.com/bedrock/home#/modelaccess) (defaults to Amazon Nova 2 Lite; choose Claude Opus 4.6 for best quality)
2. **Create an EC2 key pair** in your target region (optional, for SSH access)

## Option 1: Standalone (Single User)

One stack = one OpenClaw instance with its own VPC. Best for individual use or quick evaluation.

### Deploy via Console

1. Open CloudFormation in your preferred region:
   - [US West (Oregon)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create)
   - [US East (Virginia)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create)
   - [EU (Ireland)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create)
   - [Asia Pacific (Tokyo)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create)
2. Choose **Upload a template file**, select `openclaw-standalone.yaml`
3. Stack name: `openclaw-bedrock`
4. Configure parameters (model, instance type, etc.)
5. Check **"I acknowledge that AWS CloudFormation might create IAM resources"**
6. Click **Create stack** and wait ~8 minutes for `CREATE_COMPLETE`

### Deploy via CLI

```bash
git clone https://github.com/ddpie/openclaw-on-aws.git
cd openclaw-on-aws

aws cloudformation create-stack \
  --stack-name openclaw-bedrock \
  --template-body file://openclaw-standalone.yaml \
  --parameters \
    ParameterKey=InstanceType,ParameterValue=t4g.medium \
  --capabilities CAPABILITY_IAM
```

### After Deployment

> 💡 **Prerequisites**: Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and [Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

```bash
# 1. Get instance ID
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-bedrock \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)

# 2. Port forwarding (keep this terminal open)
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'

# 3. Get access token (open a new terminal)
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartInteractiveCommand \
  --parameters command='["sudo -u ubuntu cat /home/ubuntu/.openclaw/openclaw.json | grep token"]'

# 4. Open http://localhost:18789 in your browser and enter the token

# Or SSH into the instance directly for setup
aws ssm start-session --target $INSTANCE_ID
sudo su - ubuntu
openclaw onboard
```

## Option 2: Shared VPC (Multi-User)

One shared network + N per-user instances. Each user gets an independent EC2, IAM Role, and EBS volume.

### Step 1 — Deploy Shared VPC (once)

1. Open CloudFormation in your preferred region:
   - [US West (Oregon)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create)
   - [US East (Virginia)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create)
   - [EU (Ireland)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create)
   - [Asia Pacific (Tokyo)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create)
2. Upload `vpc-shared.yaml`, stack name: `openclaw-shared-vpc`
3. Wait for `CREATE_COMPLETE` (~2 minutes)
4. Go to **Outputs** tab, note: `VpcId`, `PublicSubnet1Id`, `PublicSubnet1AZ`

### Step 2 — Deploy Per-User Instances

1. Create a new stack, upload `openclaw-per-user.yaml`
2. Stack name: `openclaw-<username>` (e.g. `openclaw-alice`)
3. Fill in parameters:
   - `VpcId` — from Step 1 Outputs
   - `SubnetId` — from Step 1 Outputs (`PublicSubnet1Id`)
   - `AvailabilityZone` — from Step 1 Outputs (`PublicSubnet1AZ`)
   - `InstanceType` — `t4g.medium` (minimum)
4. Check **"I acknowledge that AWS CloudFormation might create IAM resources"**
5. Wait for `CREATE_COMPLETE` (~8 minutes)
6. Repeat for each user

### Batch Deploy via CLI

```bash
git clone https://github.com/ddpie/openclaw-on-aws.git
cd openclaw-on-aws

# Step 1: Shared VPC
aws cloudformation create-stack \
  --stack-name openclaw-shared-vpc \
  --template-body file://vpc-shared.yaml

# Wait for completion, then get outputs
VPC_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-shared-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)
SUBNET_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-shared-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`PublicSubnet1Id`].OutputValue' --output text)
AZ=$(aws cloudformation describe-stacks \
  --stack-name openclaw-shared-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`PublicSubnet1AZ`].OutputValue' --output text)

# Step 2: Per-user instances
for USER in alice bob charlie; do
  aws cloudformation create-stack \
    --stack-name openclaw-$USER \
    --template-body file://openclaw-per-user.yaml \
    --parameters \
      ParameterKey=VpcId,ParameterValue=$VPC_ID \
      ParameterKey=SubnetId,ParameterValue=$SUBNET_ID \
      ParameterKey=AvailabilityZone,ParameterValue=$AZ \
      ParameterKey=InstanceType,ParameterValue=t4g.medium \
    --capabilities CAPABILITY_IAM
done
```

### Step 3 — Initialize Each Instance

```bash
# Get instance ID from stack outputs
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-alice \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' --output text)

# Port forwarding (keep this terminal open)
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'

# Get access token (open a new terminal)
aws ssm start-session --target $INSTANCE_ID \
  --document-name AWS-StartInteractiveCommand \
  --parameters command='["sudo -u ubuntu cat /home/ubuntu/.openclaw/openclaw.json | grep token"]'

# Open http://localhost:18789 and enter the token

# Or connect directly for setup
aws ssm start-session --target $INSTANCE_ID
sudo su - ubuntu
openclaw onboard
```

### Teardown

> ⚠️ Delete **all user stacks first**, then the shared VPC stack.

## Important Notes

| Item | Details |
|------|---------|
| **Min instance type** | `t4g.medium` (4GB). `t4g.small` (2GB) causes OOM on Gateway startup. |
| **Subnet capacity** | ~251 instances per /24 subnet. Add subnets if needed. |
| **VPC Endpoints** | When disabled, Bedrock/SSM traffic goes over public internet. Enable for production. |
| **Termination Protection** | Enable on shared VPC stack to prevent accidental deletion. |
| **Post-deploy setup** | SSM into each instance → `sudo su - ubuntu` → `openclaw onboard` |

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   AWS Account                    │
│  ┌────────────────────────────────────────────┐  │
│  │              VPC (shared or per-stack)      │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │  │
│  │  │  User A  │  │  User B  │  │  User C  │ │  │
│  │  │ EC2+IAM  │  │ EC2+IAM  │  │ EC2+IAM  │ │  │
│  │  │  +EBS    │  │  +EBS    │  │  +EBS    │ │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘ │  │
│  │       │              │              │       │  │
│  │  ┌────┴──────────────┴──────────────┴────┐  │  │
│  │  │         VPC Endpoints (optional)       │  │  │
│  │  │   Bedrock · SSM · SSMMessages · S3     │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────┘  │
│                       │                          │
│              ┌────────┴────────┐                 │
│              │  Amazon Bedrock │                 │
│              │  Claude · Nova  │                 │
│              └─────────────────┘                 │
└─────────────────────────────────────────────────┘
```

## License

MIT
