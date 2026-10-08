---
name: k3s-ax-deploy
description: 在单机 k3s 集群上部署 google/ax（Agent Executor）及其硬依赖 Agent Substrate。当用户要求把 ax 部署到 k3s/自有集群、或 ax 报 actor resume 失败 / substrate Control API 不可达 / ateom worker 起不来时使用。覆盖 IPv6-only 集群、无本地 registry、feature gate 缺失等场景。
---

# 在 k3s 上部署 google/ax

`ax` 自身只是控制面。真正干活的是 **Agent Substrate**（`agent-substrate/substrate`）——
ax 把每个 Task 调度成 Substrate 的 actor 跑在 gVisor 沙箱里。所以部署顺序是：

```
mise 工具链 → 本地 registry → k3s feature gate → IPv6 出网 → Substrate → WorkerPool + ActorTemplate → ax 控制面 → 验证
```

**关键约束**：不要直接 `make deploy AX_IMAGE_REPO=...`。AX README 的默认假设是
GKE + 已装 Substrate + 公网 GCS bucket，三条在本环境都不成立。

---

## 0. 环境事实（先确认，不要假设）

```bash
ip -4 addr; ip -6 addr          # 本环境是 IPv6-only，pod CIDR = fd00:42::/64
kubectl get nodes -o wide       # k3s 单节点，无 taint
free -g; nproc; df -h /        # 内存 ~8G，磁盘 ~32G
ls /dev/kvm                     # 存在但本流程用 gVisor，不需要
```

本环境实测值（其他机器需重新确认）：

| 项 | 值 |
|---|---|
| k3s | v1.36.5+k3s1，单节点 `freestyle-vm` |
| 节点 IPv6 | `fdf5:7700::400:8fff:fe85:251a`（无 IPv4 全局地址） |
| Pod CIDR | `fd00:42::/64`（Service CIDR `fd00:43::/16`） |
| kubeconfig | `/etc/rancher/k3s/k3s.yaml`，root:root 600 |
| Substrate version label | `ate.dev/substrate-version=36d6d22`（由安装脚本生成，记下来后面 WorkerPool 要用） |

---

## 1. mise 工具链

Go 和 `ko` 都不在镜像里。substrate 的安装脚本硬依赖 `go`。

```bash
curl -fsSL https://mise.run | sh
# 让 bash -lc（非交互）也能拿到 PATH：.bashrc 顶部有非交互 return，
# 所以 profile 里也要写一份，否则脚本里 go: command not found
grep -q 'mise activate' ~/.bashrc || echo 'eval "$(/home/ubuntu/.local/bin/mise activate bash)"' >> ~/.bashrc
grep -q 'mise activate' ~/.profile || printf '\nexport PATH="$HOME/.local/bin:$PATH"\neval "$(mise activate bash)"\n' >> ~/.profile

export PATH="$HOME/.local/bin:$PATH"
mise use -g go ko
bash -lc 'go version && ko version'   # 必须用 bash -lc 验证
```

> **坑 1**：只在 `~/.bashrc` 里激活不够。Ubuntu 默认 `.bashrc` 对非交互 shell
> 提前 `return`，而 `hack/install-ate.sh`、后台 `nohup` 脚本都走非交互路径，
> 会报 `line 23: go: command not found`。`.profile` 必须也加一份。

---

## 2. 本地 registry

ko 必须往一个 registry 推镜像，而集群要从同一个 registry 拉。两个需求冲突：
宿主能访问 ≠ pod 能访问（详见坑 5）。

```bash
docker rm -f local-registry 2>/dev/null
mkdir -p ~/work/registry-data
docker run -d --restart=always --name local-registry \
  --network host \
  -v ~/work/registry-data:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5052 \
  registry:3

curl -s http://127.0.0.1:5052/v2/ -o /dev/null -w "%{http_code}\n"   # 期望 200
curl -s "http://[$(kubectl get node -o jsonpath='{.items[0].status.addresses[0].address}')]:5052/v2/" -o /dev/null -w "%{http_code}\n"
```

端口用 **5052** 而不是 5000/5001：k3s apiserver、containerd、Traefik 占着低端口，
registry 用 host 网络时冲突表现为自身反复 `bind: address already in use` 崩溃重启，
而 `ss -tlnp` 里**看不到**占用者（进程在其他 netns），非常难查。

告诉 containerd 这个镜像前缀去哪儿取：

```bash
sudo mkdir -p /etc/rancher/k3s/registries.d
printf 'mirrors:\n  "localhost:5001":\n    endpoint:\n      - "http://127.0.0.1:5052"\n' \
  | sudo tee /etc/rancher/k3s/registries.d/local.yaml
sudo systemctl restart k3s
```

注意这里 **key 写 `localhost:5001`、endpoint 指向 5052**：镜像名保持 `localhost:5001/...`
不动（这样 substrate/ax 的 manifest 不用改），由 containerd 做转发。
k3s 用 `registries.d` 的 mirror 而不是往 `/etc/containerd/certs.d` 塞 `hosts.toml`。

---

## 3. k3s feature gate

Substrate 的 podcert controller 需要 `ClusterTrustBundle` / `PodCertificateRequest`，
k3s 默认全关，缺了 `api` Service 会直接不可用。

```bash
sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<'EOF'
write-kubeconfig-mode: "644"
kube-apiserver-arg:
  - feature-gates=ClusterTrustBundle=true,ClusterTrustBundleProjection=true,PodCertificateRequest=true
  - runtime-config=certificates.k8s.io/v1beta1/clustertrustbundles=true,certificates.k8s.io/v1beta1/podcertificaterequests=true
EOF
sudo systemctl restart k3s
```

验证（返回里要有 `clustertrustbundles` 和 `podcertificaterequests`）：

```bash
kubectl get --raw /apis/certificates.k8s.io/v1beta1 | python3 -c \
  "import sys,json;print([r['name'] for r in json.load(sys.stdin)['resources']])"
```

> **重启会中断控制面**，集群内 pod 重启。本环境只有 Traefik/CoreDNS 等系统组件，可接受。
> `write-kubeconfig-mode: "644"` 顺带解决非 root 用户 `permission denied` 读 kubeconfig；
> 若 `~/.kube` 被 root 占用还要 `sudo chown ubuntu:ubuntu ~/.kube`。

---

## 4. IPv6-only pod 出网（最容易卡住的一步）

k3s 在 IPv6-only 集群里**不装** pod CIDR 的 MASQUERADE 规则，于是 pod 出网全部失败：

```
atelet: ... dial tcp: lookup storage.googleapis.com on [fd00:43::a]:53: server misbehaving
coredns: plugin/errors: _grpc_config.localhost. TXT: read udp ... i/o timeout
```

后果是连锁的：gVisor release 预热失败 → `gvisor-default` SandboxConfig 永远不 ready
→ 任何 actor 都起不来。**看到 atelet 里 "Sandbox asset prewarm failed" 就查这里。**

```bash
sudo ip6tables -t nat -I POSTROUTING 1 -s fd00:42::/64 ! -o cni0 -j MASQUERADE
```

用你的实际 Pod CIDR（`kubectl get node -o jsonpath='{.items[0].spec.podCIDR}'`）。

> **坑 2**：这条规则是运行时加的，**k3s 重启或节点重启会丢**。要持久化就写进
> systemd unit（`ExecStartPost`）或 netplan 触发的脚本，别指望它活过重启。

验证：

```bash
kubectl run dnstest --rm -i --restart=Never --image=alpine:3.20 --command -- \
  sh -c 'nslookup storage.googleapis.com && wget -qO- -T 10 https://storage.googleapis.com/ >/dev/null && echo HTTPS_OK'
```

DNS SERVFAIL → MASQUERADE 没生效。DNS 正常但 HTTPS 400 属于正常
（storage.googleapis.com 根路径就是 400），只要不是 `bad address` 就说明链路通了。

---

## 5. 安装 Agent Substrate

substrate 自带 kind 安装脚本，直接复用它的 kind overlay（rustfs 代替 GCS、
in-cluster otel/prometheus），但把 context 指向 k3s：

```bash
git clone --depth 1 https://github.com/agent-substrate/substrate ~/work/substrate
cd ~/work/substrate

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
export KUBECTL_CONTEXT=default      # k3s 的 context 名，不是 kind-kind
export KO_DOCKER_REPO=localhost:5001

hack/install-ate-kind.sh --deploy-ate-system --credential-provider='{"name":"k8s.io"}'
```

> **坑 3**：必须设 `KUBECTL_CONTEXT`。`install-ate-kind.sh` 默认打
> `kind-${KIND_CLUSTER_NAME}`，不设就报 `context "default" does not exist`
> ——这个报错文案有误导性，实际是"找不到 kind context"。

> **坑 4**：别用 `nohup ... &` 跑。前台跑 15–20 分钟（envoy 动态模块要 cargo 编译，
> ko 推 8 个镜像）。真要后台就用 `setsid ... < /dev/null &` 并显式给全 PATH，
> 否则非交互 shell 里 mise 不生效。

`KO_DOCKER_REPO=localhost:5001` 是必需的：kind 默认为它，不设会用 GCR 推然后失败。

安装完记下 substrate version（后面 WorkerPool 的 nodeSelector 要用）：

```bash
kubectl get node -L ate.dev/substrate-version
kubectl get ds -n ate-system -l app=atelet -L ate.dev/substrate-version
```

配置会记录在 `~/.cache/ate-setup/installs/default.yaml`，可用来复现参数。

---

## 6. WorkerPool + default-template ActorTemplate

AX 的 `--template=default-template --template-atespace=ax-system` 指向一个
**AX 自己不会创建**的 ActorTemplate。AX 只有在 task 声明了 `spec.image` 或 env 时
才用 `EnsureActorTemplateWithImage` 现场造一个；用默认值时直接引用这个模板。
不建它，所有 task 报 actor 创建失败。

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
cd ~/work/substrate && go build -o ~/work/bin/kubectl-ate ./cmd/kubectl-ate

~/work/bin/kubectl-ate create atespace ax-system
```

WorkerPool（namespace 用 `ax-system`，label 要和模板的 `workerSelector` 对上）：

```yaml
apiVersion: ate.dev/v1alpha1
kind: WorkerPool
metadata:
  name: ax
  namespace: ax-system
  labels:
    workload: ax          # ← 必须与 ActorTemplate 的 workerSelector.matchLabels 一致
spec:
  replicas: 2
  sandboxClass: gvisor
  workerImage: localhost:5001/ateom-gvisor:latest
  template:
    nodeSelector:
      ate.dev/substrate-version: "36d6d22"   # ← 换成你的版本号
    resources:
      limits:   { cpu: "2", memory: 2Gi }
      requests: { cpu: 500m, memory: 2Gi }
```

`ateom-gvisor` 不在 Substrate 安装时构建的镜像列表里（安装只构建 control plane），
要单独补：

```bash
KO_DOCKER_REPO=localhost:5001 ko build --base-import-paths --platform linux/amd64 ./cmd/ateom-gvisor
```

ActorTemplate —— **镜像必须钉 digest**：

```bash
DIGEST=$(curl -sI -H 'Accept: application/vnd.oci.image.index.v1+json' \
  http://127.0.0.1:5052/v2/ax-task-runner/manifests/latest \
  | grep -i docker-content-digest | awk '{print $2}' | tr -d '\r')
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
  command: [/usr/local/bin/ax-task-runner]
  wakeupProbe: { httpGet: { path: /readyz, port: 80 } }
  volumeMounts: [{ name: workspace, mountPath: /workspace }]
resources:
  limits: [{ name: cpu, quantity: "2" }, { name: memory, quantity: 2Gi }]
snapshotConfig:
  onCommit: SNAPSHOT_CONTENT_SCOPE_FULL     # FULL 才能 suspend/resume 保内存态
  storageLocation: gs://ate-snapshots/ax/     # ← 本地 rustfs，见下
sandboxConfig:
  sandboxClass: SANDBOX_CLASS_GVISOR
  configName: gvisor-default
volumes: [{ name: workspace, durableDir: {} }]
```

```bash
~/work/bin/kubectl-ate create actor-template -f ~/work/default-template.yaml
~/work/bin/kubectl-ate get workers        # 应有 2 个 WORKER_STATE_ACTIVE
```

> **坑 5**：模板里镜像写 tag 会直接被拒：
> `Invalid value: "localhost:5001/ax-task-runner:latest": must be pinned by digest (changing the image invalidates snapshots)`。
> 改 image 就得重建快照，所以 substrate 强制 digest 固定。

---

## 7. 构建并部署 ax 控制面

```bash
cd ~/work/ax
mkdir -p bin/linux_amd64
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" \
  -o bin/linux_amd64/ax-task-runner ./cmd/ax-task-runner
go build -trimpath -ldflags="-s -w" -o bin/ax ./cmd/ax
go build -trimpath -ldflags="-s -w" -o bin/ax-server ./cmd/ax-server

# task runner 镜像走 docker（Dockerfile.task-runner 装 Antigravity/Python）
docker build --platform linux/amd64 -t localhost:5001/ax-task-runner:latest -f Dockerfile.task-runner .
docker push localhost:5001/ax-task-runner:latest

# ax-server 走 ko
export KO_DOCKER_REPO=localhost:5001 KO_DEFAULTPLATFORMS=linux/amd64
ko build --base-import-paths --platform linux/amd64 ./cmd/ax-server
```

> **坑 6**：`ko build -t localhost:5001/ax-server:latest` 会报
> `repository can only contain the characters ...: ax-server:localhost:5001/ax-server`
> ——`-t` 和 `KO_DOCKER_REPO` 会叠加。让 `KO_DOCKER_REPO` 决定名字，
> 或者用 `ko resolve -f` 出 manifest 再改 image。

ax-server manifest 需要两处本地化修改，用 `ko resolve` 落地为普通 YAML 再改：

```bash
kubectl apply -f deploy/redis.yaml

KO_DOCKER_REPO=localhost:5001 KO_DEFAULTPLATFORMS=linux/amd64 \
  ko resolve -f deploy/ax-server.yaml > /tmp/ax-server.yaml

sed -i 's|value: "gs://dberkov-gke-dev3/ate-env/"|value: "gs://ate-snapshots/ax/"|' /tmp/ax-server.yaml
kubectl apply -f /tmp/ax-server.yaml
```

> **坑 7**：`AX_SNAPSHOTS_BUCKET` 默认指向作者自己的 GCS bucket
> `gs://dberkov-gke-dev3/ate-env/`，本环境没有 GCS 凭据。必须换成 kind overlay
> 建的本地 rustfs bucket `ate-snapshots`（rustfs + bucket-init Job 已在 ate-system 里）。
> `AX_SNAPSHOTS_BUCKET` **只在 ax-server 进程内**被 `BuildActorTemplate` 读取
> （`internal/substrate/client.go`），而自定义模板是在**已有的** default-template
> 基础上 clone 的，所以两处都要改，缺一个 suspend 就失败。

镜像引用 `ko resolve` 会写成 `localhost:5001/ax-server-<hash>@sha256:...`，
这个 hash 后缀是 ko 的正常行为，不要去掉。

---

## 8. 验证

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get pods -n ate-system      # ate-api-server / atelet / atenet-router / postgres / rustfs 全 Running
kubectl get pods -n ax-system       # ax-redis / ax-server Running
kubectl logs -n ate-system ds/atelet-36d6d22 | grep -i 'prewarm'   # 应出现 Sandbox assets prewarmed

cd ~/work/ax
./bin/ax get tasks                  # 表头 + 空列表即 CLI 与控制面通了
./bin/ax apply -f ~/work/ax-task.yaml
./bin/ax resume task demo-task
./bin/ax get tasks
./bin/ax ssh demo-task -- ls -la /workspace
```

最小 Task/ Workspace：

```yaml
apiVersion: ax.io/v1alpha1
kind: Workspace
metadata: { name: demo-workspace, atespace: default }
---
apiVersion: ax.io/v1alpha1
kind: Task
metadata: { name: demo-task, atespace: default }
spec:
  debug: true                        # ax ssh 需要
  workspaces: [{ name: demo-workspace, path: "/workspace" }]
  env: [{ name: DEMO_ENV, value: "active" }]
```

---

## 故障速查

| 症状 | 根因 | 处理 |
|---|---|---|
| `go: command not found`（安装脚本里） | mise 只在 `.bashrc`，非交互 shell 不加载 | `.profile` 也加 mise activate |
| `context "default" does not exist` | kind 脚本找 kind context | `export KUBECTL_CONTEXT=default` |
| `SERVFAIL` / `server misbehaving`（pod 内 DNS） | IPv6-only 集群缺 pod CIDR MASQUERADE | 坑 4 的 ip6tables 规则 |
| `Sandbox asset prewarm failed` | gVisor 下载依赖出网 | 同上，DNS 通了就好 |
| `bind: address already in use`（registry 反复重启） | 端口被占但 `ss` 查不到 | 换 5052 |
| `Get "https://kind-registry:5000/v2/" ... no such host` | kind overlay 的 `--localhost-registry-replacement` 指向 kind 专用名 | 改 atelet 参数为实际 registry 地址，或按坑 5 用 digest + `localhost:5001` 并靠 containerd mirror |
| `must be pinned by digest` | ActorTemplate 镜像用了 tag | 取 digest 钉死 |
| actor resume 报 `Internal ... workflow failed at step CallAteletRestore` | 沙箱镜像/资产拉不下来 | 依次查 registry 可达性、prewarm、pod 出网 |
| `ax ssh` 连不上 | task 缺 `spec.debug: true` | 加上 |
| pod 拉 `localhost:5001/...` ImagePullBackOff | containerd mirror 没生效 | 检查 `registries.d` 后 restart k3s |

---

## 收尾与持久化

本次部署里**重启 k3s 会丢**的东西：

1. `ip6tables` MASQUERADE 规则（坑 2）——最要紧，重启后集群会退化成不能出网。
2. Substrate 的 `36d6d22` node label 不会丢（存在 etcd）。
3. `default-template` ActorTemplate 在 postgres 里，不会丢。

要彻底可重放，把 `~/work/registry-data`、两个 YAML（`ax-workerpool.yaml`、
`default-template.yaml`）和本文档一起纳入版本管理；registry 用 docker
`--restart=always` 已能自愈。
