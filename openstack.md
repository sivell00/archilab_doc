# 1. 사전설정
## 1.1. 계정 설정
```
# root 사용자 password 설정 및 root login 활성화
passwd
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config

# sds 계정에 sudo 설정
sed -i '$a sds ALL=(ALL) NOPASSWD: ALL' /etc/sudoers
usermod -a -G sudo sds
```

## 1.2. Package 설치
```
apt update
apt install -y \
    curl \
    wget \
    git \
    conntrack \
    socat \
    chrony
systemctl enable chrony
systemctl start chrony
timedatectl set-ntp true
timedatectl set-timezone Asia/Seoul
chronyc activity -v
date
```

## 1.3. Kubekey 다운로드 및 실행
```
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.1.7 sh -
cp -rfv ./kk /usr/local/bin/kk
chmod +x /usr/local/bin/kk
kk version --show-supported-k8s
kk create config --with-kubernetes v1.31.2 --with-kubesphere v3.4.1
```

## 1.4. kubekey 설치를 위한 yaml 파일
```
cp config-sample.yaml ops-config.yaml
```

```
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: kvs
spec:
  hosts:
  - {name: hostname_1, address: ip_1_1, internalAddress: ip_1_2, user: sds, password: "password"}
  - {name: hostname_2, address: ip_2_1, internalAddress: ip_2_2, user: sds, password: "password"}
  - {name: hostname_3, address: ip_3_1, internalAddress: ip_3_2, user: sds, password: "password"}
  - {name: hostname_4, address: ip_4_1, internalAddress: ip_4_2, user: sds, password: "password"}
  - {name: hostname_5, address: ip_5_1, internalAddress: ip_5_2, user: sds, password: "password"}
  - {name: hostname_6, address: ip_6_1, internalAddress: ip_6_2, user: sds, password: "password"}
  - {name: hostname_7, address: ip_7_1, internalAddress: ip_7_2, user: sds, password: "password"}
  roleGroups:
    etcd:
    - hostname_1
    control-plane:
    - hostname_1
    worker:
    - hostname_2
    - hostname_3
    - hostname_4
    - hostname_5
    - hostname_6
    - hostname_7
```

## 1.5. ssh key 복사
```
apt install -y sshpass
ssh-keygen
sshpass -p "password" ssh-copy-id -o StrictHostKeyChecking=no sds@ip_1_1
sshpass -p "password" ssh-copy-id -o StrictHostKeyChecking=no sds@ip_2_1
sshpass -p "password" ssh-copy-id -o StrictHostKeyChecking=no sds@ip_3_1
sshpass -p "password" ssh-copy-id -o StrictHostKeyChecking=no sds@ip_4_1
sshpass -p "password" ssh-copy-id -o StrictHostKeyChecking=no sds@ip_5_1
sshpass -p "password" ssh-copy-id -o StrictHostKeyChecking=no sds@ip_6_1
sshpass -p "password" ssh-copy-id -o StrictHostKeyChecking=no sds@ip_7_1
```

## 1.6. 컨피그 파일 적용, 설치 확인
```
kk create cluster -f ops-config.yaml
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

## 1.7. kvs 대시보드 접속
http://<마스터 노드 IP>:30880
```
기본 로그인 정보:
    ID: admin
    PW: P@88w0rd (첫 로그인 후 변경 필요)
	    -> Passw0rd
```

## 1.8. Calico
```
root@control01:~/k8s# curl -L https://github.com/projectcalico/calico/releases/download/v3.28.1/calicoctl-linux-amd64 -o calicoctl
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 64.4M  100 64.4M    0     0  32.5M      0  0:00:01  0:00:01 --:--:-- 47.6M
root@control01:~/k8s# chmod +x calicoctl
root@control01:~/k8s# ./calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+------------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+----------------+-------------------+-------+------------+-------------+
| 192.168.11.102 | node-to-node mesh | up    | 2026-04-29 | Established |
| 192.168.11.104 | node-to-node mesh | up    | 2026-04-29 | Established |
| 192.168.11.105 | node-to-node mesh | up    | 2026-04-29 | Established |
| 192.168.11.106 | node-to-node mesh | up    | 2026-04-29 | Established |
| 192.168.11.107 | node-to-node mesh | up    | 2026-04-29 | Established |
| 192.168.11.103 | node-to-node mesh | up    | 2026-04-29 | Established |
| 192.168.11.108 | node-to-node mesh | up    | 2026-04-29 | Established |
+----------------+-------------------+-------+------------+-------------+

IPv6 BGP status
No IPv6 peers found.

```

## 1.9. Kubectl auto completion
```
root@control01:~/k8s# apt-get install bash-completion
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
bash-completion is already the newest version (1:2.11-8).
bash-completion set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 36 not upgraded.
root@control01:~/k8s# kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
root@control01:~/k8s# sed -i '$a complete -o default -F __start_kubectl kubectl' ~/.bashrc
```

## 1.10. krew
```
root@control01:~/k8s# set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
++ mktemp -d
+ cd /tmp/tmp.9q4RYESuJa
++ uname
++ tr '[:upper:]' '[:lower:]'
+ OS=linux
++ uname -m
++ sed -e s/x86_64/amd64/ -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/'
+ ARCH=amd64
+ KREW=krew-linux_amd64
+ curl -fsSLO https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-linux_amd64.tar.gz
+ tar zxvf krew-linux_amd64.tar.gz
./LICENSE
./krew-linux_amd64
+ ./krew-linux_amd64 install krew
Adding "default" plugin index from https://github.com/kubernetes-sigs/krew-index.git.
Updated the local copy of plugin index.
Installing plugin: krew
Installed plugin: krew
\
 | Use this plugin:
 |      kubectl krew
 | Documentation:
 |      https://krew.sigs.k8s.io/
 | Caveats:
 | \
 |  | krew is now installed! To start using kubectl plugins, you need to add
 |  | krew's installation directory to your PATH:
 |  |
 |  |   * macOS/Linux:
 |  |     - Add the following to your ~/.bashrc or ~/.zshrc:
 |  |         export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
 |  |     - Restart your shell.
 |  |
 |  |   * Windows: Add %USERPROFILE%\.krew\bin to your PATH environment variable
 |  |
 |  | To list krew commands and to get help, run:
 |  |   $ kubectl krew
 |  | For a full list of available plugins, run:
 |  |   $ kubectl krew search
 |  |
 |  | You can find documentation at
 |  |   https://krew.sigs.k8s.io/docs/user-guide/quickstart/.
 | /
/
```

## 1.11. ceph 설치 - 사전 작업
control 02에서 ceph를 설치하도록 한다.

가장 먼저 disk를 확인하자.
```
root@control01:~/k8s# lsblk -l
+ lsblk -l
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda    8:0    0  1.7T  0 disk
sdb    8:16   0  1.7T  0 disk
sdc    8:32   0  1.7T  0 disk
sdd    8:48   0  1.7T  0 disk
sde    8:64   0  1.7T  0 disk
sdf    8:80   0  1.7T  0 disk
sdg    8:96   0  1.7T  0 disk
sdg1   8:97   0    1G  0 part /boot/efi
sdg2   8:98   0  1.7T  0 part /
```

disk 초기화
```
root@control02:~# sgdisk --zap-all /dev/sda
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
root@control02:~# dd if=/dev/zero of="/dev/sda" bs=1M count=100 oflag=direct,dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.231301 s, 453 MB/s
root@control02:~# blkdiscard /dev/sda
root@control02:~# partprobe /dev/sda
root@control02:~# sgdisk --zap-all /dev/sdb
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
root@control02:~# dd if=/dev/zero of="/dev/sdb" bs=1M count=100 oflag=direct,dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.380212 s, 276 MB/s
root@control02:~# blkdiscard /dev/sdb
root@control02:~# partprobe /dev/sdb
root@control02:~# sgdisk --zap-all /dev/sdc
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
root@control02:~# dd if=/dev/zero of="/dev/sdc" bs=1M count=100 oflag=direct,dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.392358 s, 267 MB/s
root@control02:~# blkdiscard /dev/sdc
root@control02:~# partprobe /dev/sdc
root@control02:~# sgdisk --zap-all /dev/sdd
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
root@control02:~# dd if=/dev/zero of="/dev/sdd" bs=1M count=100 oflag=direct,dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.255054 s, 411 MB/s
root@control02:~# blkdiscard /dev/sdd
root@control02:~# partprobe /dev/sdd
root@control02:~# sgdisk --zap-all /dev/sde
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
root@control02:~# dd if=/dev/zero of="/dev/sde" bs=1M count=100 oflag=direct,dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.273213 s, 384 MB/s
root@control02:~# blkdiscard /dev/sde
root@control02:~# partprobe /dev/sde
root@control02:~# sgdisk --zap-all /dev/sdf
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
root@control02:~# dd if=/dev/zero of="/dev/sdf" bs=1M count=100 oflag=direct,dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.87628 s, 120 MB/s
root@control02:~# blkdiscard /dev/sdf
root@control02:~# partprobe /dev/sdf
```

# 2. Openstack
## 2.1. 헬름 차트
```
root@control01:~# mkdir osh
+ mkdir osh
root@control01:~# cd osh
+ cd osh
root@control01:~/osh# git clone https://opendev.org/openstack/openstack-helm.git
+ git clone https://opendev.org/openstack/openstack-helm.git
Cloning into 'openstack-helm'...
remote: Enumerating objects: 90062, done.
remote: Counting objects: 100% (37525/37525), done.
remote: Compressing objects: 100% (7087/7087), done.
remote: Total 90062 (delta 35997), reused 30438 (delta 30438), pack-reused 52537 (from 1)
Receiving objects: 100% (90062/90062), 16.32 MiB | 6.47 MiB/s, done.
Resolving deltas: 100% (66451/66451), done.
root@control01:~/osh# helm plugin install https://opendev.org/openstack/openstack-helm-plugin.git
+ helm plugin install https://opendev.org/openstack/openstack-helm-plugin.git
Installed plugin: osh
root@control01:~/osh/openstack-helm# export OPENSTACK_RELEASE=2025.1
+ export OPENSTACK_RELEASE=2025.1
+ OPENSTACK_RELEASE=2025.1
root@control01:~/osh/openstack-helm# export FEATURES="${OPENSTACK_RELEASE} ubuntu_noble"
+ export 'FEATURES=2025.1 ubuntu_noble'
+ FEATURES='2025.1 ubuntu_noble'
root@control01:~/osh/openstack-helm# export OVERRIDES_DIR=$(pwd)/values_overrides
++ pwd
+ export OVERRIDES_DIR=/root/osh/openstack-helm/values_overrides
+ OVERRIDES_DIR=/root/osh/openstack-helm/values_overrides
```

```
root@control01:~/osh/openstack-helm# helm dependency build  rabbitmq
+ helm dependency build rabbitmq
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  mariadb
+ helm dependency build mariadb
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  memcached
+ helm dependency build memcached
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  keystone
+ helm dependency build keystone
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  heat
+ helm dependency build heat
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  glance
+ helm dependency build glance
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  cinder
+ helm dependency build cinder
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  openvswitch
+ helm dependency build openvswitch
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  libvirt
+ helm dependency build libvirt
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  nova
+ helm dependency build nova
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  ovn
+ helm dependency build ovn
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  neutron
+ helm dependency build neutron
Saving 1 charts
Deleting outdated charts
root@control01:~/osh/openstack-helm# helm dependency build  horizon
+ helm dependency build horizon
Saving 1 charts
Deleting outdated charts
```

## 2.2. Ingress 설치
```
root@control01:~/osh/openstack-helm# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
root@control01:~/osh/openstack-helm# helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --version="4.8.3" \
    --namespace=openstack \
        --create-namespace \
    --set controller.kind=Deployment \
    --set controller.admissionWebhooks.enabled="false" \
    --set controller.scope.enabled="true" \
    --set controller.service.enabled="false" \
    --set controller.ingressClassResource.name=nginx \
    --set controller.ingressClassResource.controllerValue="k8s.io/ingress-nginx" \
    --set controller.ingressClassResource.default="false" \
    --set controller.ingressClass=nginx \
    --set controller.labels.app=ingress-api
Release "ingress-nginx" does not exist. Installing it now.
NAME: ingress-nginx
LAST DEPLOYED: Wed May  6 15:16:25 2026
NAMESPACE: openstack
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace openstack get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

## 2.3. metallb 설치
```
root@control01:~/osh/openstack-helm# helm repo add metallb https://metallb.github.io/metallb
"metallb" has been added to your repositories
root@control01:~/osh/openstack-helm# helm upgrade --install metallb metallb/metallb\
        --namespace=metallb-system \
        --create-namespace
Release "metallb" does not exist. Installing it now.
NAME: metallb
LAST DEPLOYED: Wed May  6 15:20:47 2026
NAMESPACE: metallb-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
MetalLB is now running in the cluster.

Now you can configure it via its CRs. Please refer to the metallb official docs
on how to use the CRs.
```

### 2.3.1. ip address pool 설정
```
root@control01:~/osh/openstack-helm# tee > /tmp/metallb_ipaddresspool.yaml <<EOF
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
    name: public
    namespace: metallb-system
spec:
    addresses:
    - "192.168.10.109/32"
EOF

root@control01:~/osh/openstack-helm# kubectl apply -f /tmp/metallb_ipaddresspool.yaml
ipaddresspool.metallb.io/public created
```

### 2.3.2. L2 advertise
```
root@control01:~/osh/openstack-helm# tee > /tmp/metallb_l2advertisement.yaml <<EOF
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
    name: public
    namespace: metallb-system
spec:
    ipAddressPools:
    - public
EOF
root@control01:~/osh/openstack-helm# kubectl apply -f /tmp/metallb_l2advertisement.yaml
l2advertisement.metallb.io/public created
```

### 2.3.3. 확인
```
root@control01:~/osh/openstack-helm# kubectl get ipaddresspool -n metallb-system public -o yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"metallb.io/v1beta1","kind":"IPAddressPool","metadata":{"annotations":{},"name":"public","namespace":"metallb-system"},"spec":{"addresses":["192.168.10.109/32"]}}
  creationTimestamp: "2026-05-06T06:42:51Z"
  generation: 1
  name: public
  namespace: metallb-system
  resourceVersion: "2285731"
  uid: 61b7f581-8fb7-4410-be12-00e461be8304
spec:
  addresses:
  - 192.168.10.109/32
  autoAssign: true
  avoidBuggyIPs: false
status:
  assignedIPv4: 0
  assignedIPv6: 0
  availableIPv4: 1
  availableIPv6: 0
```

## 2.4. Openstack Endpoint 설정
### 2.4.1. 설정
```
root@control01:~/osh/openstack-helm# tee > /tmp/openstack_endpoint_service.yaml <<EOF
---
kind: Service
apiVersion: v1
metadata:
  name: public-openstack
  namespace: openstack
  annotations:
    metallb.universe.tf/loadBalancerIPs: "ip"
spec:
  externalTrafficPolicy: Cluster
  type: LoadBalancer
  selector:
    app: ingress-api
  ports:
    - name: http
      port: 80
    - name: https
      port: 443
EOF
root@control01:~/osh/openstack-helm# kubectl apply -f /tmp/openstack_endpoint_service.yaml
service/public-openstack created
```

### 2.4.2. 확인
```
root@control01:~/osh/openstack-helm# kubectl get service -n openstack
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
public-openstack   LoadBalancer   10.233.42.172   <pending>     80:32220/TCP,443:31264/TCP   53s

```

















