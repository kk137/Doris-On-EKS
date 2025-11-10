# Apache Doris 存算分离架构在 AWS EKS 上的部署指南

## 目录
- [使用的配置文件](#使用的配置文件)
- [架构概述](#架构概述)
- [前置条件](#前置条件)
- [部署步骤](#部署步骤)
- [验证测试](#验证测试)
- [已知问题](#已知问题)
- [故障排查](#故障排查)

---

## 使用的配置文件

本部署指南使用以下配置文件（位于 `doris-on-eks/` 目录）：

### 必需文件
| 文件名 | 用途 | 使用步骤 |
|--------|------|----------|
| `fdb-operator-deployment.yaml` | FoundationDB Operator 部署配置 | 步骤 3 |
| `fdb-cluster.yaml` | FoundationDB 集群配置 | 步骤 4 |
| `doris-disaggregated-cluster.yaml` | Doris 存算分离集群配置 | 步骤 6 |

### 可选文件
| 文件名 | 用途 | 说明 |
|--------|------|------|
| `doris-storageclass.yaml` | 自定义存储类 | 如需自定义 EBS 配置 |
| `doris-node-config-daemonset.yaml` | 节点系统配置 | 如需调整系统参数 |
| `fdb-operator-x86.yaml` | x86 专用 FDB Operator | 备用配置 |

### 相关文档
| 文件名 | 说明 |
|--------|------|
| `doris-s3-storage-vault-setup.md` | S3 Storage Vault 完整配置指南 |
| `s3-storage-vault-solution-summary.md` | S3 问题解决总结 |
| `s3-storage-vault-error-analysis.md` | S3 错误详细分析 |

---

## 架构概述

### 部署架构
```
EKS 集群 (us-west-2)
├── ARM64 节点组 (m7g.2xlarge × 3) - Doris 主要工作负载
│   ├── Doris Frontend (FE) × 3
│   ├── Doris MetaService (MS) × 3
│   └── Doris Compute Group (CG) × 3
│
├── x86 节点组 (t3.medium × 3) - FoundationDB
│   └── FoundationDB × 6 (log × 3 + storage × 3)
│
└── 存储层
    ├── S3: doris-storage-us-west-2 (数据存储)
    └── EBS: 本地缓存和 FDB 持久化
```

### 组件说明
| 组件 | 作用 | 副本数 | 架构 |
|------|------|--------|------|
| Frontend (FE) | 查询协调、元数据管理 | 3 | ARM64 |
| MetaService (MS) | 元数据服务、连接 FDB | 3 | ARM64 |
| Compute Group | 无状态计算节点 | 3 | ARM64 |
| FoundationDB | 分布式 KV 存储 | 6 | x86 |
| S3 | 对象存储 (数据持久化) | - | - |

### 存算分离 vs 存算一体

| 特性 | 存算一体 | 存算分离 |
|------|---------|---------|
| CRD | DorisCluster | DorisDisaggregatedCluster |
| 组件 | FE + BE | FE + MS + Compute |
| 存储 | 本地磁盘 | S3 对象存储 |
| 计算节点 | 有状态 | 无状态 |
| 弹性扩缩容 | 慢 (需迁移数据) | 快 (秒级) |
| 依赖 | 无 | FoundationDB |

---

## 前置条件

### 1. 已有资源
- ✅ EKS 集群 (版本 1.33)
- ✅ EBS CSI Driver 已安装
- ✅ Doris Operator 已安装
- ✅ ARM64 节点组 (doris-arm-nodegroup)

### 2. 工具要求
```bash
aws --version      # AWS CLI
kubectl version    # Kubernetes CLI
```

### 3. 权限要求
- EKS 集群管理权限
- IAM 策略创建权限
- S3 桶创建权限

---

## 部署步骤

### 步骤 1: 创建 S3 存储桶

```bash
# 创建 S3 桶
aws s3 mb s3://doris-storage-us-west-2 --region us-west-2

# 验证
aws s3 ls | grep doris-storage
```

### 步骤 2: 创建 x86 节点组 (用于 FoundationDB)

```bash
aws eks create-nodegroup \
  --region us-west-2 \
  --cluster-name my-eks-cluster \
  --nodegroup-name fdb-x86-nodegroup \
  --node-role arn:aws:iam::YOUR_ACCOUNT_ID:role/eksNodeRole \
  --subnets subnet-xxx subnet-yyy \
  --instance-types t3.medium \
  --ami-type AL2023_x86_64_STANDARD \
  --disk-size 50 \
  --scaling-config minSize=3,maxSize=3,desiredSize=3 \
  --labels workload=fdb \
  --tags Purpose=FoundationDB,Name=fdb-x86-nodes
```

**等待节点组创建完成 (3-5 分钟)**:
```bash
# 检查状态
aws eks describe-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name fdb-x86-nodegroup \
  --region us-west-2 \
  --query 'nodegroup.status'

# 验证节点
kubectl get nodes -l workload=fdb
```

### 步骤 3: 部署 FoundationDB Operator

```bash
# 下载 Operator 配置
curl -o fdb-operator-deployment.yaml \
  https://raw.githubusercontent.com/apache/doris-operator/master/doc/examples/disaggregated/fdb/deployment.yaml

# 部署
kubectl apply -f fdb-operator-deployment.yaml

# 验证
kubectl get pods -n default | grep fdb-kubernetes-operator
```

**预期输出**:
```
fdb-kubernetes-operator-controller-manager-xxx   1/1   Running   0   1m
```

### 步骤 4: 部署 FoundationDB 集群

创建 `fdb-cluster.yaml`:

```yaml
apiVersion: apps.foundationdb.org/v1beta2
kind: FoundationDBCluster
metadata:
  name: doris-fdb
  namespace: default
spec:
  version: 7.1.67
  databaseConfiguration:
    redundancy_mode: double
  processCounts:
    cluster_controller: 1
    log: 3
    storage: 3
    stateless: -1
  processes:
    general:
      podTemplate:
        spec:
          nodeSelector:
            workload: fdb
          containers:
            - name: foundationdb
              resources:
                requests:
                  cpu: 500m
                  memory: 1Gi
              securityContext:
                runAsUser: 0
            - name: foundationdb-kubernetes-sidecar
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
              securityContext:
                runAsUser: 0
          initContainers:
            - name: foundationdb-kubernetes-init
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
              securityContext:
                runAsUser: 0
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 20Gi
  routing:
    useDNSInClusterFile: true
  sidecarContainer:
    enableLivenessProbe: true
    enableReadinessProbe: false
  useExplicitListenAddress: true
```

**部署 FDB 集群**:
```bash
kubectl apply -f fdb-cluster.yaml

# 监控部署 (约 3-5 分钟)
watch kubectl get fdb -n default
```

**等待状态变为**:
```
NAME        AVAILABLE   FULLREPLICATION   VERSION   AGE
doris-fdb   true        true              7.1.67    5m
```

**验证 FDB Pods**:
```bash
kubectl get pods -n default -l foundationdb.org/fdb-cluster-name=doris-fdb
```

**预期输出**:
```
NAME                                 READY   STATUS    AGE
doris-fdb-cluster-controller-28748   2/2     Running   5m
doris-fdb-log-22541                  2/2     Running   5m
doris-fdb-log-68547                  2/2     Running   5m
doris-fdb-log-84203                  2/2     Running   5m
doris-fdb-storage-15087              2/2     Running   5m
doris-fdb-storage-20232              2/2     Running   5m
doris-fdb-storage-33676              2/2     Running   5m
```

**获取 FDB ConfigMap** (Doris 需要):
```bash
kubectl get configmap doris-fdb-config -n default
```

### 步骤 5: 配置 S3 访问权限

#### 5.1 创建 IAM User
```bash
# 创建 User
aws iam create-user --user-name doris-s3-user --region us-west-2
```

#### 5.2 附加 S3FullAccess 策略
```bash
# 使用 AWS 托管策略 (推荐)
aws iam attach-user-policy \
  --user-name doris-s3-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --region us-west-2
```

> **⚠️ 重要**: 
> - 必须使用 `AmazonS3FullAccess` 而非自定义最小权限策略
> - 自定义策略可能缺少 multipart upload 等必要权限导致 403 错误
> - 生产环境可在充分测试后逐步收紧权限

#### 5.3 创建 Access Key
```bash
# 创建 Access Key
aws iam create-access-key --user-name doris-s3-user --output json > doris-access-key.json

# 查看凭证
cat doris-access-key.json | jq -r '.AccessKey | "AK: " + .AccessKeyId + "\nSK: " + .SecretAccessKey'
```

**⚠️ 安全提示**: 
- 妥善保管 Access Key
- 不要提交到代码仓库
- 定期轮换密钥

#### 5.4 创建 Kubernetes Secret
```bash
kubectl create secret generic doris-s3-credentials -n doris \
  --from-literal=access-key=YOUR_ACCESS_KEY_ID \
  --from-literal=secret-key=YOUR_SECRET_ACCESS_KEY

# 验证 Secret
kubectl get secret doris-s3-credentials -n doris
```

### 步骤 6: 部署 Doris 存算分离集群

创建 `doris-disaggregated-cluster.yaml`:

```yaml
apiVersion: disaggregated.cluster.doris.com/v1
kind: DorisDisaggregatedCluster
metadata:
  name: doris-disaggregated
  namespace: doris
spec:
  # MetaService 配置
  metaService:
    replicas: 3
    image: apache/doris:ms-3.0.8
    limits:
      cpu: 2
      memory: 4Gi
    requests:
      cpu: 1
      memory: 2Gi
    nodeSelector:
      workload: doris
    fdb:
      configMapNamespaceName:
        name: doris-fdb-config
        namespace: default
  
  # Frontend 配置
  feSpec:
    replicas: 3
    image: apache/doris:fe-3.0.8
    limits:
      cpu: 4
      memory: 8Gi
    requests:
      cpu: 2
      memory: 4Gi
    nodeSelector:
      workload: doris
  
  # Compute 组配置
  computeGroups:
    - uniqueId: cg1
      replicas: 3
      image: apache/doris:be-3.0.8
      limits:
        cpu: 8
        memory: 16Gi
      requests:
        cpu: 4
        memory: 8Gi
      nodeSelector:
        workload: doris
```

**部署集群**:
```bash
kubectl apply -f doris-disaggregated-cluster.yaml

# 监控部署进度
watch kubectl get dorisdisaggregatedcluster -n doris
```

**部署时间**: 约 5-10 分钟

**最终状态**:
```
NAME                  CLUSTERHEALTH   MSPHASE   FEPHASE   CGCOUNT   CGAVAILABLECOUNT
doris-disaggregated   green           Ready     Ready     1         1
```

**验证所有 Pods**:
```bash
kubectl get pods -n doris
```

**预期输出**:
```
NAME                              READY   STATUS    AGE
doris-disaggregated-ms-0          1/1     Running   10m
doris-disaggregated-ms-1          1/1     Running   10m
doris-disaggregated-ms-2          1/1     Running   10m
doris-disaggregated-fe-0          1/1     Running   10m
doris-disaggregated-fe-1          1/1     Running   10m
doris-disaggregated-fe-2          1/1     Running   10m
doris-disaggregated-cg1-0         1/1     Running   8m
doris-disaggregated-cg1-1         1/1     Running   8m
doris-disaggregated-cg1-2         1/1     Running   8m
```

---

## 配置 S3 Storage Vault

### 步骤 7: 创建 Storage Vault

部署完成后,需要配置 S3 作为持久化存储。

#### 7.1 连接到 Doris FE
```bash
kubectl exec -n doris doris-disaggregated-fe-0 -- mysql -h127.0.0.1 -P9030 -uroot
```

#### 7.2 创建 Storage Vault (使用 AK/SK)
```sql
CREATE STORAGE VAULT IF NOT EXISTS s3_vault_with_ak
PROPERTIES (
  'type'='S3',
  's3.endpoint'='s3.us-west-2.amazonaws.com',
  's3.region'='us-west-2',
  's3.bucket'='doris-storage-us-west-2',
  's3.root.path'='doris-data',
  's3.access_key'='YOUR_ACCESS_KEY_ID',
  's3.secret_key'='YOUR_SECRET_ACCESS_KEY',
  'provider'='S3',
  'use_path_style'='false'
);
```

> **注意**: 
> - 必须使用 AK/SK 方式,Doris 3.0.8 不支持 IRSA
> - endpoint 不要添加 `https://` 协议前缀
> - 使用从 Secret 获取的实际凭证

#### 7.3 设置为默认 Storage Vault
```sql
SET s3_vault_with_ak AS DEFAULT STORAGE VAULT;
```

#### 7.4 验证 Storage Vault
```sql
SHOW STORAGE VAULT;
```

**预期输出**:
```
Name              | Id | Properties                      | IsDefault
s3_vault_with_ak  | 1  | bucket: doris-storage-us-west-2 | true
```

#### 7.5 测试 S3 存储
```sql
-- 创建测试数据库
CREATE DATABASE IF NOT EXISTS test_s3;
USE test_s3;

-- 创建表 (自动使用默认 Storage Vault)
CREATE TABLE user_activity (
  user_id INT,
  activity_date DATE,
  activity_type VARCHAR(50),
  activity_count INT
)
DUPLICATE KEY(user_id, activity_date)
DISTRIBUTED BY HASH(user_id) BUCKETS 4
PROPERTIES (
  'replication_num' = '1',
  'storage_vault_name' = 's3_vault_with_ak'
);

-- 插入测试数据
INSERT INTO user_activity VALUES
(1, '2025-10-26', 'login', 5),
(2, '2025-10-26', 'purchase', 2);

-- 查询验证
SELECT * FROM user_activity;
```

#### 7.6 验证 S3 存储
```bash
# 检查 S3 中的数据文件
aws s3 ls s3://doris-storage-us-west-2/doris-data/data/ --recursive
```

应该看到 `.dat` 数据文件已创建。

---

## 验证测试

### 1. 检查服务端点
```bash
kubectl get svc -n doris
```

**关键服务**:
- `doris-disaggregated-fe`: Frontend 服务 (端口 9030)
- `doris-disaggregated-ms`: MetaService 服务 (端口 5000)
- `doris-disaggregated-cg1`: Compute 服务

### 2. 连接测试

#### 2.1 检查 Frontend 节点
```bash
kubectl run mysql-client --image=mysql:8.0 --restart=Never -n doris --rm -i --command -- \
  mysql -h doris-disaggregated-fe -P 9030 -uroot --execute="SHOW FRONTENDS\G"
```

**验证点**:
- 3 个 FE 节点全部 Alive: true
- 1 个 Master (IsMaster: true)
- 版本: doris-3.0.8-rc01

#### 2.2 检查 Compute 节点
```bash
kubectl run mysql-client --image=mysql:8.0 --restart=Never -n doris --rm -i --command -- \
  mysql -h doris-disaggregated-fe -P 9030 -uroot --execute="SHOW BACKENDS\G"
```

**验证点**:
- 3 个 Compute 节点全部 Alive: true
- compute_group_name: cg1
- 版本: doris-3.0.8-rc01

#### 2.3 检查 Compute Group
```bash
kubectl run mysql-client --image=mysql:8.0 --restart=Never -n doris --rm -i --command -- \
  mysql -h doris-disaggregated-fe -P 9030 -uroot --execute="SHOW COMPUTE GROUPS\G"
```

**预期输出**:
```
       Name: cg1
  IsCurrent: TRUE
 BackendNum: 3
```

### 3. 数据库操作测试

#### 3.1 创建数据库和表
```bash
kubectl run mysql-test --image=mysql:8.0 --restart=Never -n doris --rm -i --command -- \
  mysql -h doris-disaggregated-fe -P 9030 -uroot --execute="
CREATE DATABASE IF NOT EXISTS test_db;
USE test_db;
CREATE TABLE IF NOT EXISTS users (
  id INT,
  name VARCHAR(50),
  email VARCHAR(100),
  created_at DATETIME
) DISTRIBUTED BY HASH(id) BUCKETS 3;
"
```

#### 3.2 插入测试数据
```bash
kubectl run mysql-insert --image=mysql:8.0 --restart=Never -n doris --rm -i --command -- \
  mysql -h doris-disaggregated-fe -P 9030 -uroot --execute="
USE test_db;
INSERT INTO users VALUES 
  (1, 'Alice', 'alice@example.com', NOW()),
  (2, 'Bob', 'bob@example.com', NOW()),
  (3, 'Charlie', 'charlie@example.com', NOW());
SELECT * FROM users;
"
```

#### 3.3 查询测试
```bash
kubectl run mysql-query --image=mysql:8.0 --restart=Never -n doris --rm -i --command -- \
  mysql -h doris-disaggregated-fe -P 9030 -uroot --execute="
USE test_db;
SELECT COUNT(*) as total_users FROM users;
SELECT * FROM users WHERE id > 1;
"
```

### 4. 跨架构通信验证

```bash
# 检查 MetaService 到 FDB 的连接
kubectl logs doris-disaggregated-ms-0 -n doris | grep -i "fdb\|foundation" | tail -10

# 检查节点分布
kubectl get pods -n doris -o wide | grep -E "NAME|disaggregated"
kubectl get pods -n default -o wide | grep fdb
```

**验证点**:
- MS Pods 运行在 ARM 节点 (10.0.x.x)
- FDB Pods 运行在 x86 节点 (10.0.x.x)
- 跨架构通信正常

---

## 已知问题

### ✅ Issue 1: S3 Storage Vault 创建失败 (403 错误) - 已解决

**问题描述**:
```
ERROR 1105: pingS3 failed(put), Status Code: 403
```

**根本原因**:
- ~~Doris 3.0.8 Bug (HTTP vs HTTPS)~~ (已排除)
- **实际原因**: 自定义 IAM 策略权限不足

**解决方案** (2025-10-26 更新):
1. ✅ 使用 AWS 托管策略 `AmazonS3FullAccess` 而非自定义策略
2. ✅ 必须使用 AK/SK 方式 (IRSA 仍不支持)
3. ✅ 按照步骤 7 配置 Storage Vault

**验证状态**: 
- ✅ Storage Vault 创建成功
- ✅ 数据持久化到 S3
- ✅ 查询和聚合操作正常
- ✅ 生产环境就绪

**详细文档**: 
- [S3 Storage Vault 配置指南](doris-s3-storage-vault-setup.md)
- [问题解决总结](s3-storage-vault-solution-summary.md)
- [错误分析报告](s3-storage-vault-error-analysis.md)

**关键配置要点**:
- 使用 `AmazonS3FullAccess` 策略
- Endpoint: `s3.us-west-2.amazonaws.com` (不加协议前缀)
- 显式提供 `s3.access_key` 和 `s3.secret_key`
- 创建后设置为默认: `SET ... AS DEFAULT STORAGE VAULT`

### Issue 2: FoundationDB 不支持 ARM64

**问题**: FoundationDB 镜像仅支持 amd64 架构

**解决方案**: 创建独立的 x86 节点组运行 FDB

**影响**: 
- 增加约 $100/月成本 (3 × t3.medium)
- 跨架构通信无性能影响

---

## 故障排查

### 1. FDB Operator Init 容器失败

**症状**:
```
Init:CrashLoopBackOff
```

**原因**: FDB Operator 被调度到 ARM 节点

**解决方案**:
```bash
# 在 Operator Deployment 中添加 nodeSelector
spec:
  template:
    spec:
      nodeSelector:
        workload: fdb
```

### 2. FDB Pods Pending 状态

**症状**:
```
0/2 Pending - pod has unbound immediate PersistentVolumeClaims
```

**原因**: 没有默认 StorageClass

**解决方案**:
```bash
# 设置默认 StorageClass
kubectl patch storageclass doris-storage \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 3. Doris Compute 节点长时间 Init

**症状**:
```
Init:0/1 (超过 5 分钟)
```

**原因**: 镜像拉取慢或 Init 容器等待

**检查方法**:
```bash
kubectl describe pod doris-disaggregated-cg1-0 -n doris | grep -A 20 "Events:"
```

**解决方案**: 耐心等待,ARM64 镜像较大 (约 5-10 分钟)

### 4. 集群健康状态为 red

**症状**:
```
CLUSTERHEALTH: red
```

**检查步骤**:
```bash
# 1. 检查所有 Pods 状态
kubectl get pods -n doris

# 2. 检查 FDB 状态
kubectl get fdb -n default

# 3. 查看 Operator 日志
kubectl logs -n doris -l app.kubernetes.io/name=deployment --tail=50

# 4. 查看具体 Pod 日志
kubectl logs doris-disaggregated-ms-0 -n doris | tail -50
```

### 5. 无法连接到 Doris

**症状**:
```
ERROR 2005: Unknown MySQL server host
```

**检查服务**:
```bash
kubectl get svc -n doris
```

**正确的服务名**:
- `doris-disaggregated-fe` (不是 doris-disaggregated-fe-service)

---

### 配置文件清单

### 已创建的文件
```
doris-on-eks/
├── fdb-operator-deployment.yaml            # FDB Operator 部署
├── fdb-cluster.yaml                        # FDB 集群配置
├── doris-disaggregated-cluster.yaml        # Doris 存算分离集群
└── doris-s3-credentials (Secret)           # K8s Secret

相关文档:
├── doris-s3-storage-vault-setup.md         # S3 配置完整指南
├── s3-storage-vault-solution-summary.md    # 问题解决总结
└── s3-storage-vault-error-analysis.md      # 详细错误分析
```

### 已创建的 AWS 资源
```
IAM:
├── User: doris-s3-user
├── Policy: AmazonS3FullAccess (AWS 托管策略)
└── Access Key: AKIAVA5YLD3O65VB37NC

S3:
└── Bucket: doris-storage-us-west-2

EKS:
├── 节点组: doris-arm-nodegroup (m7g.2xlarge × 3)
└── 节点组: fdb-x86-nodegroup (t3.medium × 3)

Doris:
└── Storage Vault: s3_vault_with_ak (默认)
```

---

## 常用命令速查

### 集群管理
```bash
# 查看集群状态
kubectl get dorisdisaggregatedcluster -n doris

# 查看所有组件
kubectl get pods -n doris -o wide

# 查看服务
kubectl get svc -n doris

# 查看 FDB 状态
kubectl get fdb -n default
```

### 日志查看
```bash
# MetaService 日志
kubectl logs doris-disaggregated-ms-0 -n doris

# Frontend 日志
kubectl logs doris-disaggregated-fe-0 -n doris

# Compute 日志
kubectl logs doris-disaggregated-cg1-0 -n doris

# FDB 日志
kubectl logs doris-fdb-log-22541 -n default -c foundationdb
```

### 数据库连接
```bash
# 临时客户端
kubectl run mysql-client --image=mysql:8.0 --restart=Never -n doris --rm -it -- \
  mysql -h doris-disaggregated-fe -P 9030 -uroot

# 端口转发 (本地访问)
kubectl port-forward -n doris svc/doris-disaggregated-fe 9030:9030
mysql -h 127.0.0.1 -P 9030 -uroot
```

### 扩缩容操作
```bash
# 扩展 Compute 节点
kubectl patch dorisdisaggregatedcluster doris-disaggregated -n doris --type='json' \
  -p='[{"op": "replace", "path": "/spec/computeGroups/0/replicas", "value": 5}]'

# 添加新的 Compute Group
kubectl edit dorisdisaggregatedcluster doris-disaggregated -n doris
# 在 computeGroups 下添加:
# - uniqueId: cg2
#   replicas: 3
#   image: apache/doris:be-3.0.8
```

---

## 性能优化建议

### 1. 资源配置
- **FE**: 2-4 CPU, 4-8GB 内存
- **MS**: 1-2 CPU, 2-4GB 内存
- **Compute**: 4-8 CPU, 8-16GB 内存 (根据查询负载调整)
- **FDB**: 500m-1 CPU, 1-2GB 内存

### 2. 存储配置
- **FDB**: 20-50GB SSD (gp3)
- **Compute 缓存**: 根据热数据量配置

### 3. 网络优化
- 确保 FDB 和 Doris 在同一 VPC
- 使用 VPC Peering 或 PrivateLink 降低延迟

---

## 成本估算 (us-west-2)

| 资源 | 配置 | 月成本 (估算) |
|------|------|--------------|
| ARM 节点 | 3 × m7g.2xlarge | ~$300 |
| x86 节点 | 3 × t3.medium | ~$100 |
| EBS 存储 | ~300GB gp3 | ~$30 |
| S3 存储 | 按使用量 | 变动 |
| **总计** | | **~$430/月** |

---

## 清理资源

### 删除 Doris 集群
```bash
kubectl delete dorisdisaggregatedcluster doris-disaggregated -n doris
```

### 删除 FoundationDB
```bash
kubectl delete fdb doris-fdb -n default
kubectl delete -f fdb-operator-deployment.yaml
```

### 删除节点组
```bash
# 删除 x86 节点组
aws eks delete-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name fdb-x86-nodegroup \
  --region us-west-2

# 删除 ARM 节点组
aws eks delete-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name doris-arm-nodegroup \
  --region us-west-2
```

### 删除 S3 和 IAM 资源
```bash
# 清空并删除 S3 桶
aws s3 rm s3://doris-storage-us-west-2 --recursive
aws s3 rb s3://doris-storage-us-west-2

# 删除 Access Key
aws iam delete-access-key \
  --user-name doris-s3-user \
  --access-key-id YOUR_ACCESS_KEY_ID

# 分离策略
aws iam detach-user-policy \
  --user-name doris-s3-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# 删除 User
aws iam delete-user --user-name doris-s3-user
```

---

## 参考资料

- [Apache Doris 官方文档](https://doris.apache.org/)
- [Doris Operator GitHub](https://github.com/apache/doris-operator)
- [FoundationDB Operator](https://github.com/FoundationDB/fdb-kubernetes-operator)
- [AWS EKS 最佳实践](https://aws.github.io/aws-eks-best-practices/)
- [Doris 存算分离架构](https://doris.apache.org/docs/3.x/compute-storage-decoupled/overview/)

---

**文档版本**: 1.1  
**最后更新**: 2025-10-26  
**部署环境**: AWS EKS us-west-2, Doris 3.0.8, ARM64 + x86 混合架构  
**S3 Storage Vault**: ✅ 已配置并验证
