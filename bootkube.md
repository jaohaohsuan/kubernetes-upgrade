# bootkube

理想是用kubernetes來管理kubernetes的生命週期, 就像deployment一樣

## pre-flight-check

- 確保`/etc/kubernetes`目錄下是空的, 如果你要打掉重新裝也必須這麼做
- 確保etcd3的`/registry`被刪除, 執行`ETCDCTL_API=3 etcdctl del /registry --prefix`. 另一個暴力方法是砍etcd data目錄
- 清理全部container, 執行`docker ps -a -q | xargs docker rm -f -v`
- systemctl stop kubelet.service

以下是`kubelet.service`, 注意cni的bin與config的位置變了, 憑證也是. `--hostname-override`務必用IP, 原因是其他Pod並沒有mount `/etc/resolve.conf`.

```
[Service]
Environment="RKT_RUN_ARGS=--uuid-file-save=/var/run/kubelet-pod.uuid \
  --volume=resolv,kind=host,source=/etc/resolv.conf \
  --mount volume=resolv,target=/etc/resolv.conf \
  --volume var-lib-cni,kind=host,source=/var/lib/cni \
  --mount volume=var-lib-cni,target=/var/lib/cni \
  --volume var-log,kind=host,source=/var/log \
  --mount volume=var-log,target=/var/log"
ExecStartPre=/bin/mkdir -p /etc/kubernetes/manifests
ExecStartPre=/bin/mkdir -p /etc/kubernetes/cni/net.d
ExecStartPre=/bin/mkdir -p /etc/kubernetes/checkpoint-secrets
ExecStartPre=/bin/mkdir -p /etc/kubernetes/inactive-manifests
ExecStartPre=/bin/mkdir -p /var/lib/cni
Environment="KUBELET_IMAGE_TAG=v1.6.6_coreos.1"
ExecStartPre=/usr/bin/bash -c "grep 'certificate-authority-data' /etc/kubernetes/kubeconfig | awk '{print $2}' | base64 -d > /etc/kubernetes/ca.crt"
ExecStartPre=-/usr/bin/rkt rm --uuid-file=/var/run/kubelet-pod.uuid
ExecStart=/usr/lib/coreos/kubelet-wrapper \
  --kubeconfig=/etc/kubernetes/kubeconfig \
  --require-kubeconfig \
  --client-ca-file=/etc/kubernetes/ca.crt \
  --anonymous-auth=false \
  --cni-conf-dir=/etc/kubernetes/cni/net.d \
  --network-plugin=cni \
  --lock-file=/var/run/lock/kubelet.lock \
  --exit-on-lock-contention \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --allow-privileged \
  --hostname-override=192.168.60.202 \
  --node-labels=node-role.kubernetes.io/master \
  --register-with-taints=node-role.kubernetes.io/master=:NoSchedule \
  --cluster_dns=10.3.0.10 \
  --cluster_domain=cluster.local
ExecStop=-/usr/bin/rkt stop --uuid-file=/var/run/kubelet-pod.uuid
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target

```

如何使用`bootkube`部署, 根據以下的順序

- [render](#render)
- [start](#start)
- recover(kubernetes掛了才用到)

## render

- `--api-servers`列舉所有可能會訪問`apiserver`的方式, 內部會去掉scheme與port後拿來當憑證的SAN內容
- `--etcd-servers`如果不配置, bootkube會幫你產生`etcd.yaml`的manifest
- `--experimental-self-hosted-kubelet`尚未可用
- `--experimental-calico-network-policy`產生calico的yaml
- `asset-dir`kubernetes所有的配置與證書都在這裡, 如果重新render要確定此目錄是不存在的
- render完成後砍`flannel`, 因為我們已經有`calico`當主要的`cni`

```bash
bootkube render 
    --api-servers=https://192.168.60.202:443,https://127.0.0.1:443,https://localhost:443 \
    --etcd-servers=http://192.168.60.202:2379 --asset-dir=/tmp/b1 \
    --experimental-self-hosted-kubelet \
    --experimental-calico-network-policy
    
# 我們已經有calico, flannel砍
rm /tmp/b1/manifests/kube-flannel*

# 啟動kubelet.service之前的重要步驟, 複製kubeconfig
cp /tmp/b1/auth/kubeconfig /etc/kubernetes/

# kubectl 需要這個憑證配置
cp /tmp/b1/auth/kubeconfig ~/.kube/config
```

## start

```bash
systemctl start kubelet.service

bootkube start --asset-dir /tmp/b1

```

references

[Self-hosted Control Plane](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/self-hosted-kubernetes.md)
