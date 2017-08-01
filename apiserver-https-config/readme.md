# apiserver https config

## 1. 生成证书
``` shell
openssl genrsa -out dd_ca.key 2048
openssl req -x509 -new -nodes -key dd_ca.key -subj "/CN=YOURSITE.COM" -days 5000 -out dd_ca.crt
openssl genrsa -out dd_server.key 2048
HN=`hostname`
openssl req -new -key dd_server.key -subj "/CN=$HN" -out dd_server.csr
openssl x509 -req -in dd_server.csr -CA dd_ca.crt -CAkey dd_ca.key -CAcreateserial -out dd_server.crt -days 5000
openssl genrsa -out dd_cs_client.key 2048
openssl req -new -key dd_cs_client.key -subj "/CN=$HN" -out dd_cs_client.csr
openssl x509 -req -in dd_cs_client.csr -CA dd_ca.crt -CAkey dd_ca.key -CAcreateserial -out dd_cs_client.crt -days 5000

```

## 2. 修改master配置

修改/etc/kubernetes/apiserver

KUBE_API_ARGS="--log-dir=/var/log/kubernetes --secure-port=443 --client-ca-file=/var/run/kubernetes/dd_ca.crt --tls-private-key-file=/var/run/kubernetes/dd_server.key --tls-cert-file=/var/run/kubernetes/dd_server.crt"

修改/etc/kubernetes/controller-manager

KUBE_CONTROLLER_MANAGER_ARGS="--log-dir=/var/log/kubernetes --service-account-private-key-file=/var/run/kubernetes/server.key --root-ca-file=/var/run/kubernetes/ca.crt --master=https://centos-master:443 --kubeconfig=/etc/kubernetes/cmkubeconfig"

### 2.1.重启服务
``` shell
systemctl restart kube-apiserver
systemctl restart kube-controller-manager
curl https://centos-master:443/api/v1/nodes --cert /var/run/kubernetes/dd_cs_client.crt --key /var/run/kubernetes/dd_cs_client.key --cacert /var/run/kubernetes/dd_ca.crt
```

## 3. 修改子节点
### 3.1 复制证书

``` shell
$ openssl genrsa -out dd_kubelet_node1.key 2048
$ openssl req -new -key dd_kubelet_node1.key -subj "/CN=centos-minion-1" -out dd_kubelet_node1.csr
$ openssl x509 -req -in dd_kubelet_node1.csr -CA dd_ca.crt -CAkey dd_ca.key -CAcreateserial -out dd_kubelet_node1.crt -days 5000
scp dd_ca* root@centos-minion-n:/home
scp dd_cs* root@centos-minion-n:/home
curl https://centos-master:443/api/v1/nodes --cert /home/dd_kubelet_node1.crt --key /home/dd_kubelet_node1.key --cacert /home/dd_ca.crt
```
### 3.2 修改kubelet配置
``` shell
echo "apiVersion: v1
kind: Config
users:
- name: kubelet
  user:
    client-certificate: /home/dd_kubelet_node2.crt
    client-key: /home/dd_kubelet_node2.key
clusters:
- name: local
  cluster:
    certificate-authority: /home/dd_ca.crt
contexts:
- context:
    cluster: local
    user: kubelet
  name: my-context
current-context: my-context" > /var/lib/kubelet/kubeconfig
```
vi /etc/kubernetes/kubelet

KUBELET_API_SERVER="--api-servers=https://centos-master:443"

KUBELET_ARGS="--cluster-dns=172.16.254.218 --cluster-domain=EXAMPLE.COM --tls-cert-file=/home/dd_kubelet_node1.crt --tls-private-key-file=/home/dd_kubelet_node1.key --kubeconfig=/var/lib/kubelet/kubeconfig"

### 3.3 修改kubeproxy配置
``` shell
echo "apiVersion: v1
kind: Config
users:
- name: kubeproxy
  user:
    client-certificate: /home/dd_kubelet_node2.crt
    client-key: /home/dd_kubelet_node2.key
clusters:
- name: local
  cluster:
    certificate-authority: /home/dd_ca.crt
contexts:
- context:
    cluster: local
    user: kubeproxy
  name: my-context
current-context: my-context" > /var/lib/kubeproxy/proxykubeconfig
```
vi /etc/kubernetes/proxy

KUBE_PROXY_ARGS="--kubeconfig=/var/lib/kubeproxy/proxykubeconfig --master=https://centos-master:443"
### 3.4 重启服务
``` shell
systemctl restart kubelet
systemctl restart kube-proxy
```
## 4.注意事项
1. 注意yaml的配置项，如果单词错误是不会报错的，顶多是不起作用

2. dd_kubelet_node1只是个示例，如果有多台机器，每个机器的证书都不一样

3. 子节点的hostname一定要和证书里的CN一样