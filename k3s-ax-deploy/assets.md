# k3s + google/ax 部署用到的文件

部署过程中手写/改过的配置。都是从仓库原文件派生，改动点标了注释。

## 源文件

| 用途 | 来源 |
|---|---|
| AX | https://github.com/google/ax → `~/work/ax` |
| Agent Substrate | https://github.com/agent-substrate/substrate → `~/work/substrate` |

## `/etc/rancher/k3s/config.yaml`

Substrate 的 podcert controller 需要 ClusterTrustBundle / PodCertificateRequest，
k3s 默认关闭。`write-kubeconfig-mode` 顺带解决非 root 读 kubeconfig 的问题。

```yaml
write-kubeconfig-mode: "644"
kube-apiserver-arg:
  - feature-gates=ClusterTrustBundle=true,ClusterTrustBundleProjection=true,PodCertificateRequest=true
  - runtime-config=certificates.k8s.io/v1beta1/clustertrustbundles=true,certificates.k8s.io/v1beta1/podcertificaterequests=true
```

## `/etc/rancher/k3s/registries.d/local.yaml`

镜像名保持 `localhost:5001`，由 containerd 转发到实际端口 5052，
这样 substrate / ax 的 manifest 不需要改镜像名。

```yaml
mirrors:
  "localhost:5001":
    endpoint:
      - "http://127.0.0.1:5052"
```

## iptables（重启会丢！）

```bash
sudo ip6tables -t nat -I POSTROUTING 1 -s fd00:42::/64 ! -o cni0 -j MASQUERADE
```

IPv6-only 集群里 k3s 不给 pod CIDR 装 MASQUERADE，不加这条 pod 无法出网，
gVisor 资产预热会一直失败。**k3s 重启后必须重新加。**

## `ax-workerpool.yaml`

`ate.dev/substrate-version` 换成实际值（`kubectl get node -L ate.dev/substrate-version`）。
`workload: ax` label 必须与 `default-template.yaml` 的 `workerSelector.matchLabels` 一致。

```yaml
apiVersion: ate.dev/v1alpha1
kind: WorkerPool
metadata:
  name: ax
  namespace: ax-system
  labels:
    workload: ax
spec:
  replicas: 2
  sandboxClass: gvisor
  workerImage: localhost:5001/ateom-gvisor:latest
  template:
    nodeSelector:
      ate.dev/substrate-version: "36d6d22"
    resources:
      limits:
        cpu: "2"
        memory: 2Gi
      requests:
        cpu: 500m
        memory: 2Gi
```

## `default-template.yaml`

AX 的 `--template=default-template --template-atespace=ax-system` 指向它，
但 AX 自己不会创建。没有它所有 task 都会 actor 创建失败。

镜像必须钉 digest，否则 substrate 拒绝：

```
must be pinned by digest (changing the image invalidates snapshots)
```

```yaml
metadata:
  atespace: ax-system
  name: default-template
workerSelector:
  matchLabels:
    workload: ax
containers:
- name: guest
  image: localhost:5001/ax-task-runner@sha256:051ab80574c8ea99c437d6efbc0275c033df84fdb94d0a4b836fa57bae7e1e71
  command:
  - /usr/local/bin/ax-task-runner
  wakeupProbe:
    httpGet:
      path: /readyz
      port: 80
  volumeMounts:
  - name: workspace
    mountPath: /workspace
resources:
  limits:
  - name: cpu
    quantity: "2"
  - name: memory
    quantity: 2Gi
snapshotConfig:
  onCommit: SNAPSHOT_CONTENT_SCOPE_FULL
  storageLocation: gs://ate-snapshots/ax/
sandboxConfig:
  sandboxClass: SANDBOX_CLASS_GVISOR
  configName: gvisor-default
volumes:
- name: workspace
  durableDir: {}
```

`storageLocation` 用 kind overlay 建的本地 rustfs bucket，替掉仓库默认的
`gs://dberkov-gke-dev3/ate-env/`。

## ax-server manifest 的两处改动

`ko resolve` 落地成普通 YAML 后改，不要直接 apply ko:// 引用：

```bash
export KO_DOCKER_REPO=localhost:5001 KO_DEFAULTPLATFORMS=linux/amd64
ko resolve -f deploy/ax-server.yaml > /tmp/ax-server.yaml

# 仓库默认值指向作者的 GCS bucket，无凭据时快照全失败
sed -i 's|value: "gs://dberkov-gke-dev3/ate-env/"|value: "gs://ate-snapshots/ax/"|' /tmp/ax-server.yaml

kubectl apply -f deploy/redis.yaml
kubectl apply -f /tmp/ax-server.yaml
```

## 验证用 Task

```yaml
apiVersion: ax.io/v1alpha1
kind: Workspace
metadata:
  name: demo-workspace
  atespace: default
---
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: demo-task
  atespace: default
spec:
  debug: true
  workspaces:
    - name: demo-workspace
      path: "/workspace"
  env:
    - name: DEMO_ENV
      value: "active"
```

`examples/task.yaml` 里的 workspace 带 MCP server、技能注册表和外网 git clone，
在离线/受限环境会卡住；上面这份是最小可用版本。
