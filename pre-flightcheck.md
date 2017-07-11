# pre-flight-check

- 確保`/etc/kubernetes`目錄下是空的, 如果你要打掉重新裝也必須這麼做
- 確保etcd3的`/registry`被刪除, 執行`ETCDCTL_API=3 etcdctl del /registry --prefix`. 另一個暴力方法是砍etcd data目錄
- 清理全部container, 執行`docker ps -a -q | xargs docker rm -f -v`
- systemctl stop kubelet.service
- `rkt gc --grace-period=0`, 如果你用rkt運行kubelet
- 清理被kubernetes mount住的相關內容`mount | grep kube | awk '{print $3}' | xargs umount`

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