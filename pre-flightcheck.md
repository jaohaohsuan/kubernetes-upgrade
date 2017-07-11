# pre-flight-check

- 確保`/etc/kubernetes`目錄下是空的, 如果你要打掉重新裝也必須這麼做
- 確保etcd3的`/registry`被刪除, 執行`ETCDCTL_API=3 etcdctl del /registry --prefix`. 另一個暴力方法是砍etcd data目錄
- 清理全部container, 執行`docker ps -a -q | xargs docker rm -f -v`
- systemctl stop kubelet.service
- `rkt gc --grace-period=0`, 如果你用rkt運行kubelet
- 清理被kubernetes mount住的相關內容`mount | grep kube | awk '{print $3}' | xargs umount`
- 清理`rm -rf /var/lib/kubelet`
- 或是用`kubeadm reset`