# Ubuntu 22.04 单Master节点部署Kubernetes 1.28 + 可视化Dashboard


<!-- 第 1 页 -->

Ubuntu 22.04 单 Master 节点部署 Kubernetes 1.28 + 可视化 Dashboard
Kubernetes 要求永久关闭 Swap ，否则 kubelet 无法正常启动。
一、系统环境准备（所有节点执行）
1. 关闭 Swap 分区（必须）
```bash
# 临时关闭 Swap1
sudo swapoff -a2
 3
# 永久关闭 Swap （注释掉 fstab 中的 swap 行）4
sudo sed -i '/swap/s/^/#/' /etc/fstab5
 6
# 验证 Swap 已关闭（输出应为空）7
cat /etc/fstab | grep swap8
free -h9
2. 关闭防火墙
sudo ufw disable1
sudo systemctl stop ufw2
sudo systemctl disable ufw3
3. 加载内核模块并配置系统参数
# 加载必要的内核模块1
sudo modprobe overlay2
sudo modprobe br_netfilter3
 4
# 配置 sysctl 参数5
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf6
net.bridge.bridge-nf-call-iptables  = 17
net.bridge.bridge-nf-call-ip6tables = 18
net.ipv4.ip_forward                 = 19
EOF10
```


<!-- 第 2 页 -->

Kubernetes 1.24+ 不再默认使用 Docker 作为容器运行时，推荐使用 Containerd 。
 11
```bash
# 应用参数12
sudo sysctl --system13
4. 设置主机名和 hosts 解析
# 设置主机名（替换为你的主机名）1
sudo hostnamectl set-hostname k8s-master2
 3
# 配置 hosts 文件（添加本机 IP 和主机名映射，请将 192.168.1.100 替换为你服务器的实际内网 IP 地址）4
echo "192.168.1.100 k8s-master" | sudo tee -a /etc/hosts5
二、安装 Containerd 容器运行时
1. 安装依赖
sudo apt update && sudo apt install -y ca-certificates curl gnupg lsb-release1
2. 添加 Docker 官方 GPG 密钥和 apt 源
sudo mkdir -p /etc/apt/keyrings1
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o 
/etc/apt/keyrings/docker.gpg
2
 3
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] 
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee 
/etc/apt/sources.list.d/docker.list > /dev/null
4
3. 安装 Containerd
sudo apt update1
```


<!-- 第 3 页 -->

sudo apt install -y containerd.io2
4. 配置 Containerd
```bash
# 生成默认配置文件1
sudo containerd config default | sudo tee /etc/containerd/config.toml2
 3
# 修改配置，使用 systemd 作为 cgroup 驱动（必须）4
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml5
 6
# 重启 Containerd 并设置开机自启7
sudo systemctl restart containerd8
sudo systemctl enable containerd9
 10
# 验证 Containerd 状态11
sudo systemctl status containerd12
三、安装 Kubernetes 1.28 组件
1. 添加 Kubernetes 官方 apt 源
# 添加 Google Cloud 公钥1
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -
o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
2
 3
# 添加 Kubernetes 1.28 版本源4
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] 
https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee 
/etc/apt/sources.list.d/kubernetes.list
5
2. 安装指定版本的 kubeadm 、 kubelet 、 kubectl
sudo apt update1
sudo apt install -y kubelet=1.28.10-1.1 kubeadm=1.28.10-1.1 kubectl=1.28.10-1.12
 3
```


<!-- 第 4 页 -->

参数说明：
```bash
# 锁定版本，防止自动更新4
sudo apt-mark hold kubelet kubeadm kubectl5
 6
# 验证安装7
kubeadm version8
kubectl version --client9
四、初始化 Kubernetes Master 节点
1. 拉取所需镜像（可选，提前拉取加快初始化速度）
sudo kubeadm config images pull --kubernetes-version=1.28.101
2. 执行初始化命令
sudo kubeadm init \1
  --kubernetes-version=1.28.10 \2
  --apiserver-advertise-address=192.168.1.100 \3
  --pod-network-cidr=10.244.0.0/16 \4
  --service-cidr=10.96.0.0/12 \5
  --cri-socket=unix:///var/run/containerd/containerd.sock6
--apiserver-advertise-address ：替换为你的服务器内网 IP
--pod-network-cidr ： Pod 网络段，后续安装 Calico 网络插件需要使用这个网段
--service-cidr ： Service 网络段
3. 配置 kubectl 访问权限
mkdir -p $HOME/.kube1
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config2
sudo chown $(id -u):$(id -g) $HOME/.kube/config3
 4
# 验证 kubectl 连接5
```


<!-- 第 5 页 -->

此时你会看到 Master 节点状态为 NotReady ，这是因为还没有安装网络插件。
Calico 是 Kubernetes 最常用的网络插件之一，性能稳定且功能丰富。
默认情况下， Kubernetes 不会在 Master 节点上调度 Pod 。单节点部署需要移除这个污点：
输出 node/k8s-master untainted 表示成功。
kubectl get nodes6
五、安装 Calico 网络插件
```bash
# 下载 Calico 3.26 版本（兼容 K8s 1.28 ）1
kubectl apply -f 
https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/calico.yaml
2
 3
# 等待 Calico Pod 全部就绪（大约需要 1-2 分钟）4
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --
timeout=300s
5
kubectl wait --for=condition=ready pod -l k8s-app=calico-kube-controllers -n kube-system 
--timeout=300s
6
 7
# 再次验证节点状态（此时应该变为 Ready ）8
kubectl get nodes9
六、移除 Master 节点污点（单节点部署必须）
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane-1
七、部署 Kubernetes Dashboard 可视化界面
1. 部署 Dashboard v2.7.0 （兼容 K8s 1.28 ）
# 这步网络问题会失败1
kubectl apply -f 
https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
2
 3
# 创建本地 yaml 文件4
```


<!-- 第 6 页 -->

vim dashboard-aliyun.yaml5
2. 粘贴以下完整内容到dashboard-aliyun.yam（已替换所有镜像为阿里云）
```bash
# Copyright 2017 The Kubernetes Authors.1
# Licensed under the Apache License, Version 2.02
# 已将所有镜像替换为阿里云国内镜像： registry.cn-hangzhou.aliyuncs.com/google_containers3
 4
apiVersion: v15
kind: Namespace6
metadata:7
  name: kubernetes-dashboard8
 9
---10
apiVersion: v111
kind: ServiceAccount12
metadata:13
  labels:14
    k8s-app: kubernetes-dashboard15
  name: kubernetes-dashboard16
  namespace: kubernetes-dashboard17
 18
---19
kind: Service20
apiVersion: v121
metadata:22
  labels:23
    k8s-app: kubernetes-dashboard24
  name: kubernetes-dashboard25
  namespace: kubernetes-dashboard26
spec:27
  ports:28
    - port: 44329
      targetPort: 844330
      nodePort: 30000  # 已提前设置 NodePort 端口，无需后续修改31
  type: NodePort  # 已改为 NodePort ，直接通过 IP:30000 访问32
  selector:33
    k8s-app: kubernetes-dashboard34
 35
```


<!-- 第 7 页 -->

---36
apiVersion: v137
kind: Secret38
metadata:39
  labels:40
    k8s-app: kubernetes-dashboard41
  name: kubernetes-dashboard-certs42
  namespace: kubernetes-dashboard43
type: Opaque44
 45
---46
apiVersion: v147
kind: Secret48
type: Opaque49
metadata:50
  labels:51
    k8s-app: kubernetes-dashboard52
  name: kubernetes-dashboard-csrf53
  namespace: kubernetes-dashboard54
data:55
  csrf: ""56
 57
---58
apiVersion: v159
kind: Secret60
type: Opaque61
metadata:62
  labels:63
    k8s-app: kubernetes-dashboard64
  name: kubernetes-dashboard-key-holder65
  namespace: kubernetes-dashboard66
 67
---68
kind: ConfigMap69
apiVersion: v170
metadata:71
  labels:72
    k8s-app: kubernetes-dashboard73
  name: kubernetes-dashboard-settings74
  namespace: kubernetes-dashboard75
 76
---77


<!-- 第 8 页 -->

kind: Role78
apiVersion: rbac.authorization.k8s.io/v179
metadata:80
  labels:81
    k8s-app: kubernetes-dashboard82
  name: kubernetes-dashboard83
  namespace: kubernetes-dashboard84
rules:85
  - apiGroups: [""]86
    resources: ["secrets"]87
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", 
"kubernetes-dashboard-csrf"]
88
    verbs: ["get", "update", "delete"]89
  - apiGroups: [""]90
    resources: ["configmaps"]91
    resourceNames: ["kubernetes-dashboard-settings"]92
    verbs: ["get", "update"]93
  - apiGroups: [""]94
    resources: ["services"]95
    resourceNames: ["heapster", "dashboard-metrics-scraper"]96
    verbs: ["proxy"]97
  - apiGroups: [""]98
    resources: ["services/proxy"]99
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-
scraper", "http:dashboard-metrics-scraper"]
100
    verbs: ["get"]101
 102
---103
kind: ClusterRole104
apiVersion: rbac.authorization.k8s.io/v1105
metadata:106
  labels:107
    k8s-app: kubernetes-dashboard108
  name: kubernetes-dashboard109
rules:110
  - apiGroups: ["metrics.k8s.io"]111
    resources: ["pods", "nodes"]112
    verbs: ["get", "list", "watch"]113
 114
---115
apiVersion: rbac.authorization.k8s.io/v1116
kind: RoleBinding117
metadata:118


<!-- 第 9 页 -->

labels:119
    k8s-app: kubernetes-dashboard120
  name: kubernetes-dashboard121
  namespace: kubernetes-dashboard122
roleRef:123
  apiGroup: rbac.authorization.k8s.io124
  kind: Role125
  name: kubernetes-dashboard126
subjects:127
  - kind: ServiceAccount128
    name: kubernetes-dashboard129
    namespace: kubernetes-dashboard130
 131
---132
apiVersion: rbac.authorization.k8s.io/v1133
kind: ClusterRoleBinding134
metadata:135
  name: kubernetes-dashboard136
roleRef:137
  apiGroup: rbac.authorization.k8s.io138
  kind: ClusterRole139
  name: kubernetes-dashboard140
subjects:141
  - kind: ServiceAccount142
    name: kubernetes-dashboard143
    namespace: kubernetes-dashboard144
 145
---146
kind: Deployment147
apiVersion: apps/v1148
metadata:149
  labels:150
    k8s-app: kubernetes-dashboard151
  name: kubernetes-dashboard152
  namespace: kubernetes-dashboard153
spec:154
  replicas: 1155
  revisionHistoryLimit: 10156
  selector:157
    matchLabels:158
      k8s-app: kubernetes-dashboard159
  template:160


<!-- 第 10 页 -->

metadata:161
      labels:162
        k8s-app: kubernetes-dashboard163
    spec:164
      securityContext:165
        seccompProfile:166
          type: RuntimeDefault167
      containers:168
        - name: kubernetes-dashboard169
          # 阿里云镜像，国内极速拉取170
          image: registry.cn-hangzhou.aliyuncs.com/google_containers/dashboard:v2.7.0171
          imagePullPolicy: IfNotPresent172
          ports:173
            - containerPort: 8443174
              protocol: TCP175
          args:176
            - --auto-generate-certificates177
            - --namespace=kubernetes-dashboard178
          volumeMounts:179
            - name: kubernetes-dashboard-certs180
              mountPath: /certs181
            - mountPath: /tmp182
              name: tmp-volume183
          livenessProbe:184
            httpGet:185
              scheme: HTTPS186
              path: /187
              port: 8443188
            initialDelaySeconds: 30189
            timeoutSeconds: 30190
          securityContext:191
            allowPrivilegeEscalation: false192
            readOnlyRootFilesystem: true193
            runAsNonRoot: true194
            runAsUser: 1001195
      volumes:196
        - name: kubernetes-dashboard-certs197
          secret:198
            secretName: kubernetes-dashboard-certs199
        - name: tmp-volume200
          emptyDir: {}201
      serviceAccountName: kubernetes-dashboard202


<!-- 第 11 页 -->

nodeSelector:203
        kubernetes.io/os: linux204
      tolerations:205
        - key: node-role.kubernetes.io/master206
          effect: NoSchedule207
        - key: node-role.kubernetes.io/control-plane208
          effect: NoSchedule209
 210
---211
kind: Service212
apiVersion: v1213
metadata:214
  labels:215
    k8s-app: dashboard-metrics-scraper216
  name: dashboard-metrics-scraper217
  namespace: kubernetes-dashboard218
spec:219
  ports:220
    - port: 8000221
      targetPort: 8000222
  selector:223
    k8s-app: dashboard-metrics-scraper224
 225
---226
kind: Deployment227
apiVersion: apps/v1228
metadata:229
  labels:230
    k8s-app: dashboard-metrics-scraper231
  name: dashboard-metrics-scraper232
  namespace: kubernetes-dashboard233
spec:234
  replicas: 1235
  revisionHistoryLimit: 10236
  selector:237
    matchLabels:238
      k8s-app: dashboard-metrics-scraper239
  template:240
    metadata:241
      labels:242
        k8s-app: dashboard-metrics-scraper243
    spec:244


<!-- 第 12 页 -->

创建一个拥有集群管理员权限的用户，用于登录 Dashboard ：
      securityContext:245
        seccompProfile:246
          type: RuntimeDefault247
      containers:248
        - name: dashboard-metrics-scraper249
          # 阿里云镜像，国内极速拉取250
          image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-
scraper:v1.0.8
251
          imagePullPolicy: IfNotPresent252
          ports:253
            - containerPort: 8000254
              protocol: TCP255
          livenessProbe:256
            httpGet:257
              scheme: HTTP258
              path: /259
              port: 8000260
            initialDelaySeconds: 30261
            timeoutSeconds: 30262
          volumeMounts:263
          - mountPath: /tmp264
            name: tmp-volume265
          securityContext:266
            allowPrivilegeEscalation: false267
            readOnlyRootFilesystem: true268
            runAsNonRoot: true269
            runAsUser: 1001270
      serviceAccountName: kubernetes-dashboard271
      nodeSelector:272
        kubernetes.io/os: linux273
      tolerations:274
        - key: node-role.kubernetes.io/master275
          effect: NoSchedule276
        - key: node-role.kubernetes.io/control-plane277
          effect: NoSchedule278
      volumes:279
        - name: tmp-volume280
          emptyDir: {}281
3. 创建管理员用户并获取令牌


<!-- 第 13 页 -->

选择 " 令牌 " 登录，粘贴刚才获取的令牌即可。
```bash
# 创建管理员服务账户1
kubectl create serviceaccount admin-user -n kubernetes-dashboard2
 3
# 绑定集群管理员权限4
kubectl create clusterrolebinding admin-user-binding \5
  --clusterrole=cluster-admin \6
  --serviceaccount=kubernetes-dashboard:admin-user7
 8
# 获取登录令牌（有效期 1 小时）9
kubectl -n kubernetes-dashboard create token admin-user10
4. 访问 Dashboard
https:// 你的服务器 IP:300001
八、验证部署成功
1. 检查集群组件状态
kubectl get cs1
kubectl get pods -n kube-system2
```

![图（第13页）](assets/Ubuntu%2022.04%20%E5%8D%95Master%E8%8A%82%E7%82%B9%E9%83%A8%E7%BD%B2Kubernetes%201.28%20%2B%20%E5%8F%AF%E8%A7%86%E5%8C%96Dashboard/p13_img1.png)


<!-- 第 14 页 -->

所有 Pod 都应该处于 Running 状态。
然后通过 http:// 你的服务器 IP:NodePort 端口 访问 Nginx 页面，验证集群工作正常。
2. 测试部署一个示例应用
```bash
# 部署 Nginx 示例1
kubectl create deployment nginx --image=nginx:1.252
kubectl expose deployment nginx --port=80 --type=NodePort3
 4
# 查看 NodePort 端口5
kubectl get svc nginx6
```

![图（第14页）](assets/Ubuntu%2022.04%20%E5%8D%95Master%E8%8A%82%E7%82%B9%E9%83%A8%E7%BD%B2Kubernetes%201.28%20%2B%20%E5%8F%AF%E8%A7%86%E5%8C%96Dashboard/p14_img2.png)

![图（第14页）](assets/Ubuntu%2022.04%20%E5%8D%95Master%E8%8A%82%E7%82%B9%E9%83%A8%E7%BD%B2Kubernetes%201.28%20%2B%20%E5%8F%AF%E8%A7%86%E5%8C%96Dashboard/p14_img3.png)

![图（第14页）](assets/Ubuntu%2022.04%20%E5%8D%95Master%E8%8A%82%E7%82%B9%E9%83%A8%E7%BD%B2Kubernetes%201.28%20%2B%20%E5%8F%AF%E8%A7%86%E5%8C%96Dashboard/p14_img4.png)


<!-- 第 15 页 -->

九、常见问题解决
kubeadm init 失败：检查 Swap 是否已永久关闭， Containerd 配置是否正确，系统参数是否生效。
节点一直 NotReady ：检查 Calico Pod 是否正常运行，查看日志 kubectl logs -f -n kube-system calico-
node-xxx 。
Dashboard 无法访问：确保 kubectl proxy 正在运行，防火墙已关闭， URL 输入正确。
令牌过期：默认令牌有效期为 1 小时，过期后重新执行 kubectl -n kubernetes-dashboard create token 
admin-user 获取新令牌。
Kubernetes Dashboard 默认只在集群内部可访问可以通过 Dashboard 的 NodePort 暴露方式（无需 kubectl proxy ，直
接通过端口访问）
1.
2.
3.
4.
5.

![图（第15页）](assets/Ubuntu%2022.04%20%E5%8D%95Master%E8%8A%82%E7%82%B9%E9%83%A8%E7%BD%B2Kubernetes%201.28%20%2B%20%E5%8F%AF%E8%A7%86%E5%8C%96Dashboard/p15_img5.png)
