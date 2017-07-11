# bootkube

理想是用kubernetes來管理kubernetes的生命週期, 就像deployment一樣

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
