# k3s + google/ax 部署排错速查

按报错关键字定位。每条都对应一次真实失败，不是设想。

## `go: command not found`（在 hack/install-ate-kind.sh 里）

mise 装好了但脚本内拿不到 Go。Ubuntu 默认 `.bashrc` 对非交互 shell 提前 return，
`install-ate.sh` 走的正是非交互路径。

```bash
grep -q 'mise activate' ~/.profile || printf '\nexport PATH="$HOME/.local/bin:$PATH"\neval "$(mise activate bash)"\n' >> ~/.profile
bash -lc 'go version'    # 验证
```

---

## `Error: while reading kubeconfig: context "default" does not exist`

误导性文案。`install-ate-kind.sh` 里：

```bash
KIND_CLUSTER_NAME="${KIND_CLUSTER_NAME:-kind}"
export KUBECTL_CONTEXT="${KUBECTL_CONTEXT:-kind-${KIND_CLUSTER_NAME}}"
```

它默认打的是 `kind-kind`。你的 context 明明叫 `default`（存在），但脚本没去读它。

```bash
kubectl config get-contexts          # 确认 context 名
export KUBECTL_CONTEXT=default
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

---

## pod 内 DNS SERVFAIL：`server misbehaving`

```
nslookup storage.googleapis.com
** server can't find storage.googleapis.com: SERVFAIL
```

k3s IPv6-only 集群的 `ip6tables -t nat POSTROUTING` 里只有 docker 的规则
（`-s fd00:d0c:e12::/64 ! -o docker0 -j MASQUERADE`），**没有 pod CIDR 的**。
pod 流量出去时源地址是 ULA，路由器不回，于是 coredns 转发超时、
atelet 下载 gVisor 失败、`kubectl run alpine` 里 `wget` 报 `bad address`。

```bash
kubectl get node -o jsonpath='{.items[0].spec.podCIDR}'      # fd00:42::/64
sudo ip6tables -t nat -I POSTROUTING 1 -s fd00:42::/64 ! -o cni0 -j MASQUERADE
```

注意：这条规则**不持久**。`systemctl restart k3s` 之后就没了，集群会再次退化成
不能出网，而 atelet 日志里只会周期性地刷 prewarm 失败，很容易被当成网络抖动忽略。

---

## atelet: `Sandbox asset prewarm failed` / `gVisor release cache miss; downloading`

```
while fetching gs://gvisor/releases/...: dial tcp: lookup storage.googleapis.com
  on [fd00:43::a]:53: server misbehaving
```

这是上面 DNS 问题的下游症状。修好出网后重启 atelet 即可：

```bash
kubectl delete pod -n ate-system -l app=atelet
kubectl logs -n ate-system ds/atelet-36d6d22 --tail=20 | grep -i prewarm
# 期望：Sandbox assets prewarmed  config=gvisor-default  assets=1
```

同时它还会预热 `registry.k8s.io/pause:3.10.2` 到 layer cache；
这条也失败说明出网确实没通，不是 gVisor 源的问题。

---

## registry 反复 `bind: address already in use` 重启

```
level=fatal msg="error listening on debug interface: listen tcp :5001: bind: address already in use"
docker ps  →  local-registry   Restarting (1)
```

用 `--network host` 时，registry 的监听和宿主进程共享 netns。占用者在别的
netns 里（k3s/containderd 的端口），`ss -tlnp` 只能看到宿主 netns 的监听，
所以**查不到占用者**。本环境 5000/5001 都被占。

```bash
# 直接换端口，绕开排查
docker rm -f local-registry
mkdir -p ~/work/registry-data
docker run -d --restart=always --name local-registry --network host \
  -v ~/work/registry-data:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5052 registry:3
curl -s http://127.0.0.1:5052/v2/ -o /dev/null -w "%{http_code}\n"   # 200
```

---

## `ImagePullBackOff: localhost:5001/...`

k3s containerd 不认识 `localhost:5001` 是本地 registry。k3s 用
`/etc/rancher/k3s/registries.d/*.yaml` 的 mirror，不要去动
`/etc/containerd/certs.d`（那是 kind 的做法）。

```yaml
# /etc/rancher/k3s/registries.d/local.yaml
mirrors:
  "localhost:5001":
    endpoint:
      - "http://127.0.0.1:5052"
```

key 用镜像里的前缀 `localhost:5001`，endpoint 指向实际端口。
这样 manifest 里的镜像名不用改，只由 containerd 转发。

改完必须 `sudo systemctl restart k3s`。

---

## `Get "https://kind-registry:5000/v2/": no such host`

```
actor default/demo-task crashed: while creating "guest" OCI bundle:
  in imageCache.EnsureImage: ... lookup kind-registry on [fd00:43::a]:53: no such host
```

kind overlay 给 atelet 加了这个参数：

```yaml
# manifests/ate-install/kind/atelet/kustomization.yaml
- --localhost-registry-replacement=kind-registry:5000
```

`imagecache` 会把 `localhost:PORT/...` 重写成 `kind-registry:5000/...`
（`internal/imagecache/imagecache.go` 的 `rewriteLocalRegistry`）。
在非 kind 环境里 `kind-registry` 这个名字不存在。

两条出路，选一条：

- **保持镜像名 `localhost:5001/...` 并依赖 containerd mirror**（本 skill 采用）：
  atelet 重写后仍指向不存在的名字，所以要把 atelet 的该参数去掉或改成真实地址。
- 手工 patch atelet DaemonSet：

```bash
kubectl patch ds -n ate-system atelet-36d6d22 --type=json \
  -p '[{"op":"replace","path":"/spec/template/spec/containers/0/args/0",
       "value":"--localhost-registry-replacement=127.0.0.1:5052"}]'
```

注意 `kind-registry` 只在 `hack/create-kind-cluster.sh` 里由 docker 创建，
`install-ate-kind.sh` 不创建它——它假定你在 kind 流程里已经建过集群。
复用 kind overlay 到 k3s 时这个假设不成立。

---

## `must be pinned by digest`

```
Invalid value: "localhost:5001/ax-task-runner:latest":
  must be pinned by digest (changing the image invalidates snapshots)
```

ActorTemplate 的镜像一旦有快照就不能变，所以 substrate 强制 digest。

```bash
curl -sI -H 'Accept: application/vnd.oci.image.index.v1+json' \
  http://127.0.0.1:5052/v2/ax-task-runner/manifests/latest \
  | grep -i docker-content-digest
```

docker buildx 默认产出 manifest list，`Accept` 里要带上
`application/vnd.oci.image.index.v1+json`，否则可能拿到 v2 manifest 的 digest
和集群拉到的 index digest 对不上。

---

## `ko build -t` 报仓库名非法

```
Error: failed to publish images: repository can only contain the characters
  `abcdefghijklmnopqrstuvwxyz0123456789_-./`: ax-server:localhost:5001/ax-server
```

`-t` 指定的 tag 前缀会和 `KO_DOCKER_REPO` 叠加成 `ax-server:localhost:5001/ax-server`。

```bash
# 不要同时给 -t 和带 tag 的 KO_DOCKER_REPO
export KO_DOCKER_REPO=localhost:5001
ko build --base-import-paths --platform linux/amd64 ./cmd/ax-server
```

要拿带 ko 解析的 manifest：

```bash
ko resolve -f deploy/ax-server.yaml > /tmp/ax-server.yaml
```

`ko resolve` 生成的 image 是 `localhost:5001/ax-server-<md5>@sha256:...`，
hash 后缀正常，别手贱去掉。

---

## `ax-server` 起来但 suspend/resume 失败：`gs://dberkov-gke-dev3/ate-env/`

仓库里 `deploy/ax-server.yaml` 的默认值指向作者的 GCS bucket：

```yaml
- name: AX_SNAPSHOTS_BUCKET
  value: "gs://dberkov-gke-dev3/ate-env/"
```

无 GCS 凭据时快照读写全失败。换成 kind overlay 建的本地 rustfs：

```bash
sed -i 's|value: "gs://dberkov-gke-dev3/ate-env/"|value: "gs://ate-snapshots/ax/"|' /tmp/ax-server.yaml
```

**两处都要改**：
- ax-server 的 `AX_SNAPSHOTS_BUCKET` 环境变量（`BuildActorTemplate` 读它）
- `default-template` ActorTemplate 的 `snapshotConfig.storageLocation`

自定义模板是从 default-template clone 的（`EnsureActorTemplateWithImage` →
`BuildActorTemplate`，后者在 `snapshotsBucket == ""` 时才读环境变量），
只改一处会出现"建 task 成功但 suspend 失败"。

---

## task 报 actor 创建失败，但 WorkerPool 是 Ready

AX 的 `--template=default-template --template-atespace=ax-system` 指向一个
**AX 不会自己创建**的 ActorTemplate。只有 task 声明了 `spec.image` 或 `spec.env`
时，`EnsureActorTemplateWithImage` 才会现场造一个；用默认值时直接引用它。

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
~/work/bin/kubectl-ate get actor-templates -a ax-system   # 空 = 缺模板
~/work/bin/kubectl-ate create actor-template -f ~/work/default-template.yaml
```

`reconciler.go` 里 `--template` 支持 `atespace/name` 形式，也可以指向别的 atespace。

---

## `ax ssh` 连不上

task spec 需要 `debug: true`，它让 runner 暴露 guest services：

```yaml
spec:
  debug: true
```

---

## 集群访问卡在 coredns，`kubectl get nodes` 却正常

Node 有 Ready Condition，但 coredns 里全是：

```
plugin/errors: 2 _grpc_config.localhost. TXT: read udp [fd00:42::4]:38663->
  [2606:4700:4700::1111]:53: i/o timeout
```

这不是 DNS 配置问题，是宿主 IPv6 出网本身有问题（宿主 `/etc/resolv.conf` 里
`8.8.8.8` 那两条报 `network is unreachable`，宿主是 IPv6-only）。
先在宿主上验证，再看 pod：

```bash
curl -sI https://storage.googleapis.com/ | head -1        # 宿主可达？
kubectl run t --rm -i --restart=Never --image=alpine -- \
  sh -c 'nslookup storage.googleapis.com'                  # pod 可达？
```
