# How to run DeepFlow CI/CD with Serverless ACK

> 目标：结合当前仓库的 GitHub Actions（如 `cli-build.yml`、`docker-compose-build.yml`），给出可落地的 Serverless 方案，并说明计费归属。

## 1. 先回答核心问题：能不能用 Serverless（ACK）？

可以。推荐两种落地方式：

1. **推荐（改动小）**：保留 GitHub Actions 作为编排入口，在 Workflow 中调用 ACK Serverless 集群执行重任务（编译/镜像构建）。
2. **可选（更彻底）**：在 ACK 上部署自托管 GitHub Runner（actions-runner-controller），由 ACK 承载 Runner 计算资源。

对当前 DeepFlow 仓库来说，优先建议方案 1，因为可以最小化改动已有流程（已存在 tag 规则与发布逻辑）。

---

## 2. 计费方式说明（重点）

### 2.1 是否“合并计费”到 GitHub Actions？

**不会自动合并。**

- **GitHub Actions 计费**：按 GitHub 规则对工作流运行时长/分钟计费（或用量包）。
- **阿里云 ACK Serverless 计费**：按 ACK Serverless 的 Pod/CPU/内存/网络/存储等资源消耗计费。

两者是**独立账单体系**：

- GitHub 负责“触发与编排”成本；
- 阿里云负责“实际计算资源”成本。

### 2.2 如何理解总成本

总成本 ≈ GitHub Actions 成本 + 阿里云 ACK Serverless 成本。  
如果将重编译/重构建下沉到 ACK，通常会减少 GitHub Runner 压力，但不会让 GitHub 账单变为 0。

---

## 3. 与当前仓库 CI/CD 的结合点

当前仓库中：

- `/.github/workflows/cli-build.yml`
- `/.github/workflows/docker-compose-build.yml`

已经包含：

- tag 触发逻辑；
- `develop-local` 祖先检查；
- Release/OSS 上传行为。

因此落地时建议：

1. 保留现有触发与发布逻辑；
2. 将“重计算步骤”迁移到 ACK Job；
3. GitHub Actions 仅做：触发、提交 ACK Job、等待完成、收集产物并发布。

---

## 4. 落地方案（推荐）：GitHub Actions + ACK Serverless Job

## 4.1 架构

1. 开发者 push/tag（现有 workflow 触发）。
2. GitHub Actions 使用阿里云凭证访问 ACK。
3. Workflow 通过 `kubectl apply` 提交 Kubernetes Job 到 ACK Serverless。
4. Job 在集群内完成构建并上传到 OSS（或产物仓库）。
5. Workflow 轮询 Job 状态，成功后继续 Release。

## 4.2 前置准备

1. 阿里云创建 ACK Serverless 集群（建议单独命名如 `deepflow-ci-ack`）。
2. 为 CI 创建最小权限 RAM 子账号，授予：
   - ACK 访问；
   - OSS（按需要）；
   - 容器镜像仓库（按需要）。
3. 在 GitHub 仓库配置 Secrets（示例）：
   - `ALIYUN_ACCESS_KEY_ID`
   - `ALIYUN_ACCESS_KEY_SECRET`
   - `ALIYUN_REGION`
   - `ACK_CLUSTER_ID`
4. 准备 kubeconfig（或通过阿里云 CLI 动态拉取）。

## 4.3 Workflow 示例（可直接改造）

以下示例展示如何在现有 workflow 中加入 ACK Job 执行阶段（示意片段）：

```yaml
- name: Setup kubectl
  uses: azure/setup-kubectl@v4

- name: Install aliyun cli
  run: |
    curl -fsSL https://aliyuncli.alicdn.com/aliyun-cli-linux-latest-amd64.tgz | tar -xz
    sudo mv aliyun /usr/local/bin/aliyun

- name: Configure aliyun
  run: |
    aliyun configure set \
      --profile akProfile \
      --mode AK \
      --region "${{ secrets.ALIYUN_REGION }}" \
      --access-key-id "${{ secrets.ALIYUN_ACCESS_KEY_ID }}" \
      --access-key-secret "${{ secrets.ALIYUN_ACCESS_KEY_SECRET }}"

- name: Build kubeconfig for ACK
  run: |
    aliyun cs GET /k8s/${{ secrets.ACK_CLUSTER_ID }}/user_config > kubeconfig.json
    cat kubeconfig.json | jq -r '.config' > $HOME/.kube/config

- name: Submit ACK build job
  run: |
    kubectl apply -f .github/ack/cli-build-job.yaml

- name: Wait ACK build job done
  env:
    ACK_JOB_NAME: deepflow-cli-build
  run: |
    kubectl wait --for=condition=complete --timeout=3600s job/${ACK_JOB_NAME}
```

> 建议将 Job YAML 放在 `/.github/ack/`，便于与 workflow 配置统一管理。

## 4.4 ACK Job 示例（`cli-build-job.yaml`）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: deepflow-cli-build
spec:
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: builder
          image: golang:1.24
          command: ["/bin/sh","-c"]
          args:
            - |
              set -e
              git clone --depth=1 https://github.com/JiezengDev/deepflow.git /workspace/deepflow
              cd /workspace/deepflow/cli
              make
              # 可在此上传 OSS，或写入对象存储
```

---

## 5. 可选方案：ACK 上运行自托管 GitHub Runner

适合希望将大部分 CI 计算能力统一放到云上管理的团队。

技术路径：

1. ACK 部署 `actions-runner-controller`；
2. 创建 RunnerDeployment（或 autoscaling runner set）；
3. workflow 的 `runs-on` 切换为对应 self-hosted label。

优点：workflow 改动更少；缺点：维护复杂度更高（Runner 生命周期、安全基线、镜像维护）。

---

## 6. 安全与稳定性建议

1. **最小权限**：RAM 只给 ACK/OSS 必要权限。
2. **短期凭证优先**：可用 OIDC + STS 替代长期 AK。
3. **网络控制**：ACK 命名空间隔离、出网策略限制。
4. **构建可重复**：固定构建镜像版本，避免“今天可过、明天失败”。
5. **失败可观测**：`kubectl logs` 与 GitHub job summary 输出统一。

---

## 7. 迁移建议（分阶段）

1. **Phase 1（1~2 天）**：仅把 `cli-build` 的编译下沉 ACK Job，发布逻辑不改。
2. **Phase 2（1~2 天）**：将 `docker-compose` 打包也下沉 ACK。
3. **Phase 3（按需）**：评估 server/agent 重任务是否切换到 ACK 或 self-hosted runner。

---

## 8. 结论

- DeepFlow 当前 CI/CD 可以平滑接入阿里云 ACK Serverless。
- 计费不会并入 GitHub Actions，而是 GitHub 与阿里云分别计费。
- 推荐先采用“GitHub Actions 编排 + ACK Job 执行”的低风险方案，逐步迁移重任务，保证现有 tag/release 规则稳定可用。
