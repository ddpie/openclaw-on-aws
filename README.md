# OpenClaw on AWS

> Deploy OpenClaw on AWS with Amazon Bedrock. No API keys. IAM authentication. ~$13/month per user.

[English](#english) | [中文](#中文)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![AWS](https://img.shields.io/badge/AWS-Bedrock-orange.svg)](https://aws.amazon.com/bedrock/)
[![CloudFormation](https://img.shields.io/badge/IaC-CloudFormation-blue.svg)](https://aws.amazon.com/cloudformation/)

---

<a id="english"></a>

## Deployment Options

| | Standalone | Shared VPC (Multi-User) |
|---|---|---|
| **Use case** | Single user | Teams (10–100+ users) |
| **Template** | `openclaw-standalone.yaml` | `vpc-shared.yaml` + `openclaw-per-user.yaml` |
| **VPC** | One per stack | Shared across all users |
| **Isolation** | Full network isolation | IAM + EBS isolation, shared network |

## Prerequisites

1. **Enable Bedrock models** in the [Bedrock Console](https://console.aws.amazon.com/bedrock/) (recommended: Claude Opus 4.6)
2. **Create an EC2 key pair** in your target region

## Option 1: Standalone (Single User)

One stack = one OpenClaw instance with its own VPC. Best for individual use or quick evaluation.

### Launch Stack

| Region | Launch |
|--------|--------|
| **US West (Oregon)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |
| **US East (Virginia)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |
| **EU (Ireland)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |
| **Asia Pacific (Tokyo)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |

### After Deployment

```bash
# Get instance ID
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-bedrock \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)

# Connect via SSM
aws ssm start-session --target $INSTANCE_ID

# Switch to ubuntu user
sudo su - ubuntu

# Verify
openclaw gateway status

# Initialize
openclaw onboard
```

## Option 2: Shared VPC (Multi-User)

One shared network + N per-user instances. Each user gets an independent EC2, IAM Role, and EBS volume.

### Step 1 — Deploy Shared VPC (once)

| Region | Launch |
|--------|--------|
| **US West (Oregon)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |
| **US East (Virginia)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |
| **EU (Ireland)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |
| **Asia Pacific (Tokyo)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |

Note the Outputs: `VpcId`, `PublicSubnet1Id`, `PublicSubnet1AZ`.

### Step 2 — Deploy Per-User Instances

| Region | Launch |
|--------|--------|
| **US West (Oregon)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |
| **US East (Virginia)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |
| **EU (Ireland)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |
| **Asia Pacific (Tokyo)** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |

Fill in `VpcId`, `SubnetId`, and `AvailabilityZone` from the shared VPC stack outputs.

### Batch Deploy via CLI

```bash
VPC_ID=vpc-xxx
SUBNET_ID=subnet-xxx
AZ=us-west-2a

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

---

<a id="中文"></a>

# OpenClaw on AWS（中文）

> 在 AWS 上部署 OpenClaw，使用 Amazon Bedrock。无需 API Key，IAM 认证，每人每月约 $13。

## 部署方案

| | 独立部署 | 共享 VPC（多人） |
|---|---|---|
| **适用场景** | 单人使用 | 团队（10–100+ 人） |
| **模板** | `openclaw-standalone.yaml` | `vpc-shared.yaml` + `openclaw-per-user.yaml` |
| **VPC** | 每人独立 | 全员共享 |
| **隔离性** | 完全网络隔离 | IAM + 数据卷隔离，网络共享 |

## 前置条件

1. 在 [Bedrock 控制台](https://console.aws.amazon.com/bedrock/) **开启模型访问**（推荐 Claude Opus 4.6）
2. 在目标 Region **创建 EC2 密钥对**

## 方案一：独立部署（单人）

一个 Stack = 一套完整的 OpenClaw（含独立 VPC）。适合个人使用或快速体验。

### 一键部署

| Region | 部署 |
|--------|------|
| **美西（Oregon）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |
| **美东（Virginia）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |
| **欧洲（Ireland）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |
| **亚太（Tokyo）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create/review?stackName=openclaw-bedrock&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-standalone.yaml) |

### 部署后操作

```bash
# 获取实例 ID
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-bedrock \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)

# SSM 登录
aws ssm start-session --target $INSTANCE_ID

# 切换到 ubuntu 用户
sudo su - ubuntu

# 验证状态
openclaw gateway status

# 初始化
openclaw onboard
```

## 方案二：共享 VPC（多人部署）

一套共享网络 + N 个用户实例。每人独立 EC2、IAM Role、EBS 数据卷。

### 第一步 — 部署共享网络（仅一次）

| Region | 部署 |
|--------|------|
| **美西（Oregon）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |
| **美东（Virginia）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |
| **欧洲（Ireland）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |
| **亚太（Tokyo）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create/review?stackName=openclaw-shared-vpc&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/vpc-shared.yaml) |

记录 Outputs 中的 `VpcId`、`PublicSubnet1Id`、`PublicSubnet1AZ`。

### 第二步 — 部署用户实例

| Region | 部署 |
|--------|------|
| **美西（Oregon）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |
| **美东（Virginia）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |
| **欧洲（Ireland）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |
| **亚太（Tokyo）** | [![Launch](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/create/review?stackName=openclaw-user&templateURL=https://raw.githubusercontent.com/ddpie/openclaw-on-aws/main/openclaw-per-user.yaml) |

填入共享 VPC Stack 输出的 `VpcId`、`SubnetId`、`AvailabilityZone`。

### 批量部署脚本

```bash
VPC_ID=vpc-xxx
SUBNET_ID=subnet-xxx
AZ=us-west-2a

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

## 许可证

MIT
