# install storageOS in Kubernetes 

需要装key/value storage, 目前官方用的是[consul](https://www.consul.io/), docker运行环境版本为`1.12.6`, kubernetes `v1.7.0`

```bash
ip=`hostname -I`
num_nodes=1
leader_ip=`hostname -I`
docker run -d --name consul --net=host consul agent -server -bind=${ip} -client=0.0.0.0 -bootstrap-expect=${num_nodes} -retry-join=${leader_ip}
```

启动storageos server

```bash
mkdir /var/lib/storageos
wget -O /etc/docker/plugins/storageos.json https://docs.storageos.com/assets/storageos.json
modprobe nbd nbds_max=1024

docker run -d --name storageos \
    -e HOSTNAME \
    -e ADVERTISE_IP=192.168.60.200 \
    --net=host --pid=host --privileged --cap-add SYS_ADMIN \
    --device /dev/fuse \
    -v /var/lib/storageos:/var/lib/storageos:rshared \
    -v /run/docker/plugins:/run/docker/plugins \
    storageos/node server
```

添加storageOS的storageClass的账号密码与连接地址到kububernetes的secret

```bash
kubectl create secret generic storageos-secret \
    --type="kubernetes.io/storageos" \
    --from-literal=apiAddress=tcp://localhost:5705 \
    --from-literal=apiUsername=storageos \
    --from-literal=apiPassword=storageos \
    --namespace=default
```

创建storageOS的`StorageClass`

```bash
cat << EOF > fast-storageos.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: fast
provisioner: kubernetes.io/storageos
parameters:
  pool: default
  description: Kubernetes volume
  fsType: ext4
  adminSecretNamespace: default
  adminSecretName: storageos-secret
EOF

kubectl apply -f fast-storageos.yaml
```

创建一个pvc测试

```bash
cat << EOF > fast0001.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast0001
  annotations:
    volume.beta.kubernetes.io/storage-class: fast
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f fast0001.yaml
```

clean up

```bash
kubectl delete -f fast0001.yaml
kubectl delete -f fast-storageos.yaml
```