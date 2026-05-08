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
# image download가 실패해서 cn의 mirror를 사용하도록 한다.
export KKZONE=cn
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
root@control01:~/osh/openstack-helm# kubectl create namespace openstack
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
    metallb.universe.tf/loadBalancerIPs: "192.168.10.109"
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
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
public-openstack   LoadBalancer   10.233.42.172   192.168.10.109   80:32220/TCP,443:31264/TCP   2m25s
```

## 2.5. Node Label 설정
openstack control palne : control01
openstack network infra : control03
ceph : control02
cpu compute : cpu01, cpu02, cpu03
gpu compyte : gpu01, gpu02
```
root@control01:~# kubectl get nodes
NAME        STATUS   ROLES           AGE     VERSION
control01   Ready    control-plane   7d16h   v1.31.2
control02   Ready    worker          7d16h   v1.31.2
control03   Ready    worker          7d16h   v1.31.2
cpu01       Ready    worker          7d16h   v1.31.2
cpu02       Ready    worker          7d16h   v1.31.2
cpu03       Ready    worker          7d16h   v1.31.2
gpu01       Ready    worker          7d16h   v1.31.2
gpu02       Ready    worker          7d16h   v1.31.2
root@control01:~# kubectl label --overwrite nodes control01 openstack-control-plane=enabled
node/control01 labeled
root@control01:~# kubectl label --overwrite nodes control03 openstack-network-node=enabled
node/control03 labeled
root@control01:~# kubectl label --overwrite nodes cpu01 cpu02   cpu03 gpu01 gpu02 linuxbridge=enabled
node/cpu01 labeled
node/cpu02 labeled
node/cpu03 labeled
node/gpu01 labeled
node/gpu02 labeled
root@control01:~# kubectl label --overwrite nodes cpu01 cpu02   cpu03 gpu01 gpu02 openvswitch=enabled
node/cpu01 labeled
node/cpu02 labeled
node/cpu03 labeled
node/gpu01 labeled
node/gpu02 labeled
root@control01:~# kubectl label --overwrite nodes cpu01 cpu02   cpu03 gpu01 gpu02 openstack-compute-node=enabled
node/cpu01 labeled
node/cpu02 labeled
node/cpu03 labeled
node/gpu01 labeled
node/gpu02 labeled
root@control01:~# kubectl label --overwrite nodes control03 l3-agent=enabled
node/control03 labeled
root@control01:~# kubectl label --overwrite nodes control02 ceph-osd=enabled
node/control02 labeled
root@control01:~# kubectl label --overwrite nodes control02 ceph-mds=enabled
node/control02 labeled
root@control01:~# kubectl label --overwrite nodes control02 ceph-node=enabled
node/control02 labeled
```

## 2.6. ceph-rook.sh 스크립트 수정
원래 하라는 거...
```
> 43 라인
nodeSelector:
  ceph-node: enabled

> 241 라인
아래 항목 삭제, device filter 추가 (서버에 장착된 Disk별로 다름)
    devices:
      - name: "${CEPH_OSD_DATA_DEVICE}"
        config:
          databaseSizeMB: "5120"
          walSizeMB: "2048"

deviceFilter: "^sd[a-c]"
```

```
root@control01:~/osh/openstack-helm# sed -n '40,45p' tools/deployment/ceph/ceph-rook.sh
  pullPolicy: IfNotPresent
crds:
  enabled: true
nodeSelector:
  ceph-node: enabled
tolerations: []
root@control01:~/osh/openstack-helm# sed -n '235,245p' tools/deployment/ceph/ceph-rook.sh
    mon: system-node-critical
    osd: system-node-critical
    mgr: system-cluster-critical
  storage: # cluster level storage configuration and selection
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[a-f]"
  disruptionManagement:
    managePodBudgets: true
    osdMaintenanceTimeout: 30
    pgHealthCheckTimeout: 0
```




```
root@control01:~/osh/openstack-helm# ./tools/deployment/ceph/ceph-rook.sh
+ ROOK_RELEASE=v1.19.3
+ : /dev/loop100
+ :
+ '[' -s /tmp/ceph-fs-uuid.txt ']'
+ uuidgen
++ cat /tmp/ceph-fs-uuid.txt
+ CEPH_FS_ID=99f59474-da8a-4a6f-814c-a27f8b5cb238
+ . /etc/os-release
++ PRETTY_NAME='Ubuntu 24.04.4 LTS'
++ NAME=Ubuntu
++ VERSION_ID=24.04
++ VERSION='24.04.4 LTS (Noble Numbat)'
++ VERSION_CODENAME=noble
++ ID=ubuntu
++ ID_LIKE=debian
++ HOME_URL=https://www.ubuntu.com/
++ SUPPORT_URL=https://help.ubuntu.com/
++ BUG_REPORT_URL=https://bugs.launchpad.net/ubuntu/
++ PRIVACY_POLICY_URL=https://www.ubuntu.com/legal/terms-and-policies/privacy-policy
++ UBUNTU_CODENAME=noble
++ LOGO=ubuntu-logo
+ '[' xubuntu == xcentos ']'
+ '[' xubuntu == xubuntu ']'
++ uname -r
+ dpkg --compare-versions 6.8.0-110-generic lt 4.5
+ CRUSH_TUNABLES=null
+ tee /tmp/rook.yaml
image:
  repository: rook/ceph
  tag: v1.19.3
  pullPolicy: IfNotPresent
crds:
  enabled: true
nodeSelector:
  ceph-node: enabled
tolerations: []
unreachableNodeTolerationSeconds: 5
currentNamespaceOnly: false
annotations: {}
logLevel: INFO
rbacEnable: true
pspEnable: false
priorityClassName:
allowLoopDevices: true
csi:
  enableRbdDriver: true
  enableCephfsDriver: false
  enableGrpcMetrics: false
  enableCSIHostNetwork: true
  enableCephfsSnapshotter: true
  enableNFSSnapshotter: true
  enableRBDSnapshotter: true
  enablePluginSelinuxHostMount: false
  enableCSIEncryption: false
  pluginPriorityClassName: system-node-critical
  provisionerPriorityClassName: system-cluster-critical
  rbdFSGroupPolicy: "File"
  cephFSFSGroupPolicy: "File"
  nfsFSGroupPolicy: "File"
  enableOMAPGenerator: false
  cephFSKernelMountOptions:
  enableMetadata: false
  provisionerReplicas: 1
  clusterName: ceph
  logLevel: 0
  sidecarLogLevel:
  rbdPluginUpdateStrategy:
  rbdPluginUpdateStrategyMaxUnavailable:
  cephFSPluginUpdateStrategy:
  nfsPluginUpdateStrategy:
  grpcTimeoutInSeconds: 150
  allowUnsupportedVersion: false
  csiRBDPluginVolume:
  csiRBDPluginVolumeMount:
  csiCephFSPluginVolume:
  csiCephFSPluginVolumeMount:
  provisionerTolerations:
  provisionerNodeAffinity: #key1=value1,value2; key2=value3
  pluginTolerations:
  pluginNodeAffinity: # key1=value1,value2; key2=value3
  enableLiveness: false
  cephfsGrpcMetricsPort:
  cephfsLivenessMetricsPort:
  rbdGrpcMetricsPort:
  csiAddonsPort:
  forceCephFSKernelClient: true
  rbdLivenessMetricsPort:
  kubeletDirPath:
  cephcsi:
    image:
  registrar:
    image:
  provisioner:
    image:
  snapshotter:
    image:
  attacher:
    image:
  resizer:
    image:
  imagePullPolicy: IfNotPresent
  cephfsPodLabels: #"key1=value1,key2=value2"
  nfsPodLabels: #"key1=value1,key2=value2"
  rbdPodLabels: #"key1=value1,key2=value2"
  csiAddons:
    enabled: false
    image: "quay.io/csiaddons/k8s-sidecar:v0.5.0"
  nfs:
    enabled: false
  topology:
    enabled: false
    domainLabels:
  readAffinity:
    enabled: false
    crushLocationLabels:
  cephFSAttachRequired: true
  rbdAttachRequired: true
  nfsAttachRequired: true
enableDiscoveryDaemon: false
cephCommandsTimeoutSeconds: "15"
useOperatorHostNetwork:
discover:
  toleration:
  tolerationKey:
  tolerations:
  nodeAffinity: # key1=value1,value2; key2=value3
  podLabels: # "key1=value1,key2=value2"
  resources:
disableAdmissionController: true
hostpathRequiresPrivileged: false
disableDeviceHotplug: false
discoverDaemonUdev:
imagePullSecrets:
enableOBCWatchOperatorNamespace: true
admissionController:
+ helm repo add rook-release https://charts.rook.io/release
"rook-release" has been added to your repositories
+ helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph --v                                                ersion v1.19.3 -f /tmp/rook.yaml
NAME: rook-ceph
LAST DEPLOYED: Thu May  7 10:52:40 2026
NAMESPACE: rook-ceph
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Rook Operator has been installed.

Visit https://rook.io/docs/rook/latest for instructions on how to create and configure Rook                                                 clusters

Important Notes:
- You must customize the 'CephCluster' resource in the sample manifests for your cluster.
- Each CephCluster must be deployed to its own namespace, the samples use `rook-ceph` for th                                                e namespace.
- The sample manifests assume you also installed the rook-ceph operator in the `rook-ceph` n                                                amespace.
- The helm chart includes all the RBAC required to create a CephCluster CRD in the same name                                                space.
- Any disk devices you add to the cluster in the 'CephCluster' must be empty (no filesystem                                                 and no partitions).
- The CSI operator will manage the CSI driver lifecycle for RBD, CephFS, and NFS drivers.
+ helm osh wait-for-pods rook-ceph
+ tee /tmp/ceph.yaml
operatorNamespace: rook-ceph
clusterName: ceph
kubeVersion:
configOverride: |
  [global]
  mon_allow_pool_delete = true
  mon_allow_pool_size_one = true
  osd_pool_default_size = 1
  osd_pool_default_min_size = 1
  mon_warn_on_pool_no_redundancy = false
  auth_allow_insecure_global_id_reclaim = false
toolbox:
  enabled: true
  tolerations: []
  affinity: {}
  resources:
    limits:
      cpu: "100m"
      memory: "64Mi"
    requests:
      cpu: "100m"
      memory: "64Mi"
  priorityClassName:
monitoring:
  enabled: false
  metricsDisabled: true
  createPrometheusRules: false
  rulesNamespaceOverride:
  prometheusRule:
    labels: {}
    annotations: {}
pspEnable: false
cephClusterSpec:
  cephVersion:
    image: quay.io/ceph/ceph:v20.2.1
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  skipUpgradeChecks: false
  continueUpgradeAfterChecksEvenIfNotHealthy: false
  waitTimeoutForHealthyOSDInMinutes: 10
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 3
    allowMultiplePerNode: false
    modules:
      - name: pg_autoscaler
        enabled: true
      - name: dashboard
        enabled: false
      - name: nfs
        enabled: false
  dashboard:
    enabled: true
    ssl: true
  network:
    connections:
      encryption:
        enabled: false
      compression:
        enabled: false
      requireMsgr2: false
    provider: host
  crashCollector:
    disable: true
  logCollector:
    enabled: true
    periodicity: daily # one of: hourly, daily, weekly, monthly
    maxLogSize: 500M # SUFFIX may be 'M' or 'G'. Must be at least 1M.
  cleanupPolicy:
    confirmation: ""
    sanitizeDisks:
      method: quick
      dataSource: zero
      iteration: 1
    allowUninstallWithVolumes: false
  monitoring:
    enabled: false
    metricsDisabled: true

  removeOSDsIfOutAndSafeToRemove: false
  priorityClassNames:
    mon: system-node-critical
    osd: system-node-critical
    mgr: system-cluster-critical
  storage: # cluster level storage configuration and selection
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[a-f]"
  disruptionManagement:
    managePodBudgets: true
    osdMaintenanceTimeout: 30
    pgHealthCheckTimeout: 0
  healthCheck:
    daemonHealth:
      mon:
        disabled: false
        interval: 45s
      osd:
        disabled: false
        interval: 60s
      status:
        disabled: false
        interval: 60s
    livenessProbe:
      mon:
        disabled: false
      mgr:
        disabled: false
      osd:
        disabled: false
ingress:
  dashboard:
    {}
cephBlockPools:
  - name: rbd
    namespace: ceph
    spec:
      failureDomain: host
      replicated:
        size: 1
    storageClass:
      enabled: true
      name: general
      isDefault: true
      reclaimPolicy: Delete
      allowVolumeExpansion: true
      volumeBindingMode: "Immediate"
      mountOptions: []
      allowedTopologies: []
      parameters:
        imageFormat: "2"
        imageFeatures: layering
        csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/provisioner-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/controller-expand-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
        csi.storage.k8s.io/node-stage-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/fstype: ext4
cephFileSystems: []
# Not needed in general for openstack-helm. Uncomment if needed.
# cephFileSystems:
#   - name: cephfs
#     namespace: ceph
#     spec:
#       metadataPool:
#         replicated:
#           size: 1
#       dataPools:
#         - failureDomain: host
#           replicated:
#             size: 1
#           name: data
#       metadataServer:
#         activeCount: 1
#         activeStandby: false
#         priorityClassName: system-cluster-critical
#     storageClass:
#       enabled: true
#       isDefault: false
#       name: ceph-filesystem
#       pool: data0
#       reclaimPolicy: Delete
#       allowVolumeExpansion: true
#       volumeBindingMode: "Immediate"
#       mountOptions: []
#       parameters:
#         csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
#         csi.storage.k8s.io/provisioner-secret-namespace: "{{ .Release.Namespace }}"
#         csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
#         csi.storage.k8s.io/controller-expand-secret-namespace: "{{ .Release.Namespace }}"
#         csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
#         csi.storage.k8s.io/node-stage-secret-namespace: "{{ .Release.Namespace }}"
#         csi.storage.k8s.io/fstype: ext4
cephBlockPoolsVolumeSnapshotClass:
  enabled: false
  name: general
  isDefault: false
  deletionPolicy: Delete
  annotations: {}
  labels: {}
  parameters: {}
cephObjectStores:
  - name: default
    namespace: ceph
    spec:
      allowUsersInNamespaces:
        - "*"
      metadataPool:
        failureDomain: host
        replicated:
          size: 1
      dataPool:
        failureDomain: host
        replicated:
          size: 1
      preservePoolsOnDelete: true
      gateway:
        port: 8080
        instances: 1
        priorityClassName: system-cluster-critical
    storageClass:
      enabled: true
      name: ceph-bucket
      reclaimPolicy: Delete
      volumeBindingMode: "Immediate"
      parameters:
        region: us-east-1
+ helm upgrade --install --create-namespace --namespace ceph rook-ceph-cluster --set operato                                                rNamespace=rook-ceph rook-release/rook-ceph-cluster --version v1.19.3 -f /tmp/ceph.yaml
Release "rook-ceph-cluster" does not exist. Installing it now.
NAME: rook-ceph-cluster
LAST DEPLOYED: Thu May  7 10:53:26 2026
NAMESPACE: ceph
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Ceph Cluster has been installed. Check its status by running:
  kubectl --namespace ceph get cephcluster

Visit https://rook.io/docs/rook/latest/CRDs/Cluster/ceph-cluster-crd/ for more information a                                                bout the Ceph CRD.

Important Notes:
- You can only deploy a single cluster per namespace
- If you wish to delete this cluster and start fresh, you will also have to wipe the OSD dis                                                ks using `sfdisk`
+ helm osh wait-for-pods rook-ceph
+ kubectl wait --namespace=ceph --for=condition=ready pod --selector=app=rook-ceph-tools --t                                                imeout=600s
pod/rook-ceph-tools-665997cbf8-v48gx condition met
++ date +%s
+ wait_start_time=1778119025
++ date +%s
+ [[ 0 -lt 1800 ]]
+ sleep 30
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-mon --no-headers
++ awk '{ print $1 }'
+ MON_PODS=rook-ceph-mon-c-canary-7cbc5d6d4c-9cxzj
++ echo rook-ceph-mon-c-canary-7cbc5d6d4c-9cxzj
++ wc -w
+ MON_PODS_NUM=1
+ MON_PODS_READY=0
+ for MON_POD in $MON_PODS
+ kubectl get pod --namespace=ceph rook-ceph-mon-c-canary-7cbc5d6d4c-9cxzj
+ kubectl wait --namespace=ceph --for=condition=ready pod/rook-ceph-mon-c-canary-7cbc5d6d4c-9cxzj --timeout=60s
pod/rook-ceph-mon-c-canary-7cbc5d6d4c-9cxzj condition met
+ MON_PODS_READY=1
+ [[ 1 == 1 ]]
+ echo 'Monitor pods are ready. Moving on.'
Monitor pods are ready. Moving on.
+ break
+ echo '=========== CEPH K8S PODS LIST ============'
=========== CEPH K8S PODS LIST ============
+ kubectl get pods -n rook-ceph -o wide
NAME                                                     READY   STATUS              RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
ceph-csi-controller-manager-6567d74759-hkw76             1/1     Running             0          5m5s    10.233.91.4      gpu01       <none>           <none>
rook-ceph-operator-bf7b7fc44-m7whm                       1/1     Running             0          5m5s    10.233.116.5     control02   <none>           <none>
rook-ceph.rbd.csi.ceph.com-ctrlplugin-79cd9bdf45-7lgjz   5/5     Running             0          2m58s   10.233.91.6      gpu01       <none>           <none>
rook-ceph.rbd.csi.ceph.com-nodeplugin-2qwhh              0/2     ContainerCreating   0          2m58s   192.168.11.108   gpu02       <none>           <none>
rook-ceph.rbd.csi.ceph.com-nodeplugin-56q2w              2/2     Running             0          2m58s   192.168.11.103   control03   <none>           <none>
rook-ceph.rbd.csi.ceph.com-nodeplugin-9m99p              2/2     Running             0          2m58s   192.168.11.104   cpu01       <none>           <none>
rook-ceph.rbd.csi.ceph.com-nodeplugin-dz85n              2/2     Running             0          2m58s   192.168.11.107   gpu01       <none>           <none>
rook-ceph.rbd.csi.ceph.com-nodeplugin-ggb79              2/2     Running             0          2m58s   192.168.11.102   control02   <none>           <none>
rook-ceph.rbd.csi.ceph.com-nodeplugin-s86p2              2/2     Running             0          2m58s   192.168.11.105   cpu02       <none>           <none>
rook-ceph.rbd.csi.ceph.com-nodeplugin-v2gs6              2/2     Running             0          2m58s   192.168.11.106   cpu03       <none>           <none>
+ kubectl get pods -n ceph -o wide
NAME                                      READY   STATUS        RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
rook-ceph-mon-c-canary-7cbc5d6d4c-9cxzj   2/2     Terminating   0          3m      192.168.11.108   gpu02   <none>           <none>
rook-ceph-tools-665997cbf8-v48gx          1/1     Running       0          4m24s   192.168.11.108   gpu02   <none>           <none>
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-rgw --no-headers
++ awk '{print $1; exit}'
No resources found in ceph namespace.
+ RGW_POD=
++ date +%s
+ wait_start_time=1778119072
+ [[ -z '' ]]
++ date +%s
+ [[ 0 -lt 1800 ]]
+ sleep 30
+ date '+%Y-%m-%d %H:%M:%S'
2026-05-07 10:58:22
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-tools --no-headers
++ grep Running
++ awk '{ print $1; exit }'
+ TOOLS_POD=rook-ceph-tools-665997cbf8-v48gx
+ [[ -z rook-ceph-tools-665997cbf8-v48gx ]]
+ echo '=========== CEPH STATUS ============'
=========== CEPH STATUS ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph -s
  cluster:
    id:     bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 7s) [leader: a]
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

+ echo '=========== CEPH OSD POOL LIST ============'
=========== CEPH OSD POOL LIST ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph osd pool ls
+ echo '=========== CEPH K8S PODS LIST ============'
=========== CEPH K8S PODS LIST ============
+ kubectl get pods -n ceph -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
rook-ceph-mgr-a-66bbc6dc4b-5rv24   2/3     Running   0          4s      192.168.11.105   cpu02   <none>           <none>
rook-ceph-mgr-b-94bd58b6f-lqfwq    2/3     Running   0          4s      192.168.11.107   gpu01   <none>           <none>
rook-ceph-mgr-c-9bddbd7b4-qrbz6    2/3     Running   0          4s      192.168.11.108   gpu02   <none>           <none>
rook-ceph-mon-a-f5c4df79d-tbb8r    2/2     Running   0          31s     192.168.11.107   gpu01   <none>           <none>
rook-ceph-mon-b-795ffcd64d-jvx55   2/2     Running   0          27s     192.168.11.105   cpu02   <none>           <none>
rook-ceph-mon-c-7d65b7f68c-nrt9t   1/2     Running   0          17s     192.168.11.108   gpu02   <none>           <none>
rook-ceph-tools-665997cbf8-v48gx   1/1     Running   0          4m59s   192.168.11.108   gpu02   <none>           <none>
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-rgw --no-headers
++ awk '{print $1; exit}'
No resources found in ceph namespace.
+ RGW_POD=
+ [[ -z '' ]]
++ date +%s
+ [[ 35 -lt 1800 ]]
+ sleep 30
+ date '+%Y-%m-%d %H:%M:%S'
2026-05-07 10:58:57
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-tools --no-headers
++ grep Running
++ awk '{ print $1; exit }'
+ TOOLS_POD=rook-ceph-tools-665997cbf8-v48gx
+ [[ -z rook-ceph-tools-665997cbf8-v48gx ]]
+ echo '=========== CEPH STATUS ============'
=========== CEPH STATUS ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph -s
  cluster:
    id:     bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 44s) [leader: a]
    mgr: a(active, since 11s), standbys: b, c
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

+ echo '=========== CEPH OSD POOL LIST ============'
=========== CEPH OSD POOL LIST ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph osd pool ls
+ echo '=========== CEPH K8S PODS LIST ============'
=========== CEPH K8S PODS LIST ============
+ kubectl get pods -n ceph -o wide
NAME                                    READY   STATUS            RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
rook-ceph-mgr-a-66bbc6dc4b-5rv24        3/3     Running           0          43s     192.168.11.105   cpu02       <none>           <none>
rook-ceph-mgr-b-94bd58b6f-lqfwq         3/3     Running           0          43s     192.168.11.107   gpu01       <none>           <none>
rook-ceph-mgr-c-9bddbd7b4-qrbz6         3/3     Running           0          43s     192.168.11.108   gpu02       <none>           <none>
rook-ceph-mon-a-f5c4df79d-tbb8r         2/2     Running           0          70s     192.168.11.107   gpu01       <none>           <none>
rook-ceph-mon-b-795ffcd64d-jvx55        2/2     Running           0          66s     192.168.11.105   cpu02       <none>           <none>
rook-ceph-mon-c-7d65b7f68c-nrt9t        2/2     Running           0          56s     192.168.11.108   gpu02       <none>           <none>
rook-ceph-osd-prepare-control02-t8nbh   0/1     PodInitializing   0          19s     10.233.116.6     control02   <none>           <none>
rook-ceph-osd-prepare-control03-wvldw   0/1     PodInitializing   0          21s     10.233.73.8      control03   <none>           <none>
rook-ceph-osd-prepare-cpu01-vlxdc       0/1     PodInitializing   0          21s     10.233.79.2      cpu01       <none>           <none>
rook-ceph-osd-prepare-cpu02-w2r8s       0/1     Completed         0          20s     10.233.126.7     cpu02       <none>           <none>
rook-ceph-osd-prepare-cpu03-gngwt       0/1     PodInitializing   0          20s     10.233.121.2     cpu03       <none>           <none>
rook-ceph-osd-prepare-gpu01-vvs46       0/1     Completed         0          19s     10.233.91.7      gpu01       <none>           <none>
rook-ceph-osd-prepare-gpu02-brm8v       0/1     Completed         0          19s     10.233.112.6     gpu02       <none>           <none>
rook-ceph-tools-665997cbf8-v48gx        1/1     Running           0          5m38s   192.168.11.108   gpu02       <none>           <none>
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-rgw --no-headers
++ awk '{print $1; exit}'
No resources found in ceph namespace.
+ RGW_POD=
+ [[ -z '' ]]
++ date +%s
+ [[ 74 -lt 1800 ]]
+ sleep 30
+ date '+%Y-%m-%d %H:%M:%S'
2026-05-07 10:59:36
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-tools --no-headers
++ grep Running
++ awk '{ print $1; exit }'
+ TOOLS_POD=rook-ceph-tools-665997cbf8-v48gx
+ [[ -z rook-ceph-tools-665997cbf8-v48gx ]]
+ echo '=========== CEPH STATUS ============'
=========== CEPH STATUS ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph -s
  cluster:
    id:     bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 1

  services:
    mon: 3 daemons, quorum a,b,c (age 84s) [leader: a]
    mgr: a(active, since 25s), standbys: b, c
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

+ echo '=========== CEPH OSD POOL LIST ============'
=========== CEPH OSD POOL LIST ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph osd pool ls
+ echo '=========== CEPH K8S PODS LIST ============'
=========== CEPH K8S PODS LIST ============
+ kubectl get pods -n ceph -o wide
NAME                                    READY   STATUS            RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
rook-ceph-mgr-a-66bbc6dc4b-5rv24        3/3     Running           0          83s     192.168.11.105   cpu02       <none>           <none>
rook-ceph-mgr-b-94bd58b6f-lqfwq         3/3     Running           0          83s     192.168.11.107   gpu01       <none>           <none>
rook-ceph-mgr-c-9bddbd7b4-qrbz6         3/3     Running           0          83s     192.168.11.108   gpu02       <none>           <none>
rook-ceph-mon-a-f5c4df79d-tbb8r         2/2     Running           0          110s    192.168.11.107   gpu01       <none>           <none>
rook-ceph-mon-b-795ffcd64d-jvx55        2/2     Running           0          106s    192.168.11.105   cpu02       <none>           <none>
rook-ceph-mon-c-7d65b7f68c-nrt9t        2/2     Running           0          96s     192.168.11.108   gpu02       <none>           <none>
rook-ceph-osd-prepare-control02-t8nbh   0/1     PodInitializing   0          59s     10.233.116.6     control02   <none>           <none>
rook-ceph-osd-prepare-control03-wvldw   0/1     PodInitializing   0          61s     10.233.73.8      control03   <none>           <none>
rook-ceph-osd-prepare-cpu01-vlxdc       0/1     PodInitializing   0          61s     10.233.79.2      cpu01       <none>           <none>
rook-ceph-osd-prepare-cpu02-w2r8s       0/1     Completed         0          60s     10.233.126.7     cpu02       <none>           <none>
rook-ceph-osd-prepare-cpu03-gngwt       0/1     PodInitializing   0          60s     10.233.121.2     cpu03       <none>           <none>
rook-ceph-osd-prepare-gpu01-vvs46       0/1     Completed         0          59s     10.233.91.7      gpu01       <none>           <none>
rook-ceph-osd-prepare-gpu02-brm8v       0/1     Completed         0          59s     10.233.112.6     gpu02       <none>           <none>
rook-ceph-tools-665997cbf8-v48gx        1/1     Running           0          6m18s   192.168.11.108   gpu02       <none>           <none>
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-rgw --no-headers
++ awk '{print $1; exit}'
No resources found in ceph namespace.
+ RGW_POD=
+ [[ -z '' ]]
++ date +%s
+ [[ 114 -lt 1800 ]]
+ sleep 30
+ date '+%Y-%m-%d %H:%M:%S'
2026-05-07 11:00:16
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-tools --no-headers
++ grep Running
++ awk '{ print $1; exit }'
+ TOOLS_POD=rook-ceph-tools-665997cbf8-v48gx
+ [[ -z rook-ceph-tools-665997cbf8-v48gx ]]
+ echo '=========== CEPH STATUS ============'
=========== CEPH STATUS ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph -s
  cluster:
    id:     bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 2m) [leader: a]
    mgr: a(active, since 65s), standbys: b, c
    osd: 8 osds: 2 up (since 5s), 8 in (since 13s)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   121 MiB used, 3.5 TiB / 3.5 TiB avail
    pgs:     100.000% pgs unknown
             1 unknown

+ echo '=========== CEPH OSD POOL LIST ============'
=========== CEPH OSD POOL LIST ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph osd pool ls
.mgr
default.rgw.control
rbd
+ echo '=========== CEPH K8S PODS LIST ============'
=========== CEPH K8S PODS LIST ============
+ kubectl get pods -n ceph -o wide
NAME                                    READY   STATUS      RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
rook-ceph-mgr-a-66bbc6dc4b-5rv24        3/3     Running     0          2m2s    192.168.11.105   cpu02       <none>           <none>
rook-ceph-mgr-b-94bd58b6f-lqfwq         3/3     Running     0          2m2s    192.168.11.107   gpu01       <none>           <none>
rook-ceph-mgr-c-9bddbd7b4-qrbz6         3/3     Running     0          2m2s    192.168.11.108   gpu02       <none>           <none>
rook-ceph-mon-a-f5c4df79d-tbb8r         2/2     Running     0          2m29s   192.168.11.107   gpu01       <none>           <none>
rook-ceph-mon-b-795ffcd64d-jvx55        2/2     Running     0          2m25s   192.168.11.105   cpu02       <none>           <none>
rook-ceph-mon-c-7d65b7f68c-nrt9t        2/2     Running     0          2m15s   192.168.11.108   gpu02       <none>           <none>
rook-ceph-osd-0-b8ff77fcd-xf5sk         1/2     Running     0          11s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-1-6555ccdbc9-55xws        1/2     Running     0          20s     192.168.11.103   control03   <none>           <none>
rook-ceph-osd-2-5798984758-xr4vg        1/2     Running     0          11s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-3-575ff49f7d-h9km7        1/2     Running     0          20s     192.168.11.103   control03   <none>           <none>
rook-ceph-osd-4-7c65755ccd-66fcm        1/2     Running     0          11s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-5-7d8786698-w6ll9         1/2     Running     0          11s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-6-85bfb9cd4c-bvmtb        1/2     Running     0          11s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-7-5d78b94bb4-6dbz9        1/2     Running     0          11s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-prepare-control02-t8nbh   0/1     Completed   0          98s     10.233.116.6     control02   <none>           <none>
rook-ceph-osd-prepare-control03-wvldw   0/1     Completed   0          100s    10.233.73.8      control03   <none>           <none>
rook-ceph-osd-prepare-cpu01-vlxdc       0/1     Completed   0          100s    10.233.79.2      cpu01       <none>           <none>
rook-ceph-osd-prepare-cpu02-w2r8s       0/1     Completed   0          99s     10.233.126.7     cpu02       <none>           <none>
rook-ceph-osd-prepare-cpu03-gngwt       0/1     Completed   0          99s     10.233.121.2     cpu03       <none>           <none>
rook-ceph-osd-prepare-gpu01-vvs46       0/1     Completed   0          98s     10.233.91.7      gpu01       <none>           <none>
rook-ceph-osd-prepare-gpu02-brm8v       0/1     Completed   0          98s     10.233.112.6     gpu02       <none>           <none>
rook-ceph-tools-665997cbf8-v48gx        1/1     Running     0          6m57s   192.168.11.108   gpu02       <none>           <none>
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-rgw --no-headers
++ awk '{print $1; exit}'
No resources found in ceph namespace.
+ RGW_POD=
+ [[ -z '' ]]
++ date +%s
+ [[ 153 -lt 1800 ]]
+ sleep 30
+ date '+%Y-%m-%d %H:%M:%S'
2026-05-07 11:00:55
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-tools --no-headers
++ grep Running
++ awk '{ print $1; exit }'
+ TOOLS_POD=rook-ceph-tools-665997cbf8-v48gx
+ [[ -z rook-ceph-tools-665997cbf8-v48gx ]]
+ echo '=========== CEPH STATUS ============'
=========== CEPH STATUS ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph -s
  cluster:
    id:     bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 2m) [leader: a]
    mgr: a(active, since 104s), standbys: b, c
    osd: 8 osds: 8 up (since 37s), 8 in (since 53s)
    rgw: 1 daemon active (1 hosts, 1 zones)

  data:
    pools:   10 pools, 59 pgs
    objects: 227 objects, 589 KiB
    usage:   216 MiB used, 14 TiB / 14 TiB avail
    pgs:     59 active+clean

  io:
    client:   228 KiB/s rd, 10 KiB/s wr, 289 op/s rd, 148 op/s wr

+ echo '=========== CEPH OSD POOL LIST ============'
=========== CEPH OSD POOL LIST ============
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph osd pool ls
.mgr
default.rgw.control
rbd
default.rgw.meta
default.rgw.log
default.rgw.buckets.index
default.rgw.buckets.non-ec
default.rgw.otp
.rgw.root
default.rgw.buckets.data
+ echo '=========== CEPH K8S PODS LIST ============'
=========== CEPH K8S PODS LIST ============
+ kubectl get pods -n ceph -o wide
NAME                                      READY   STATUS      RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
rook-ceph-mgr-a-66bbc6dc4b-5rv24          3/3     Running     0          2m41s   192.168.11.105   cpu02       <none>           <none>
rook-ceph-mgr-b-94bd58b6f-lqfwq           3/3     Running     0          2m41s   192.168.11.107   gpu01       <none>           <none>
rook-ceph-mgr-c-9bddbd7b4-qrbz6           3/3     Running     0          2m41s   192.168.11.108   gpu02       <none>           <none>
rook-ceph-mon-a-f5c4df79d-tbb8r           2/2     Running     0          3m8s    192.168.11.107   gpu01       <none>           <none>
rook-ceph-mon-b-795ffcd64d-jvx55          2/2     Running     0          3m4s    192.168.11.105   cpu02       <none>           <none>
rook-ceph-mon-c-7d65b7f68c-nrt9t          2/2     Running     0          2m54s   192.168.11.108   gpu02       <none>           <none>
rook-ceph-osd-0-b8ff77fcd-xf5sk           2/2     Running     0          50s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-1-6555ccdbc9-55xws          2/2     Running     0          59s     192.168.11.103   control03   <none>           <none>
rook-ceph-osd-2-5798984758-xr4vg          2/2     Running     0          50s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-3-575ff49f7d-h9km7          2/2     Running     0          59s     192.168.11.103   control03   <none>           <none>
rook-ceph-osd-4-7c65755ccd-66fcm          2/2     Running     0          50s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-5-7d8786698-w6ll9           2/2     Running     0          50s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-6-85bfb9cd4c-bvmtb          2/2     Running     0          50s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-7-5d78b94bb4-6dbz9          2/2     Running     0          50s     192.168.11.102   control02   <none>           <none>
rook-ceph-osd-prepare-control02-t8nbh     0/1     Completed   0          2m17s   10.233.116.6     control02   <none>           <none>
rook-ceph-osd-prepare-control03-wvldw     0/1     Completed   0          2m19s   10.233.73.8      control03   <none>           <none>
rook-ceph-osd-prepare-cpu01-vlxdc         0/1     Completed   0          2m19s   10.233.79.2      cpu01       <none>           <none>
rook-ceph-osd-prepare-cpu02-w2r8s         0/1     Completed   0          2m18s   10.233.126.7     cpu02       <none>           <none>
rook-ceph-osd-prepare-cpu03-gngwt         0/1     Completed   0          2m18s   10.233.121.2     cpu03       <none>           <none>
rook-ceph-osd-prepare-gpu01-vvs46         0/1     Completed   0          2m17s   10.233.91.7      gpu01       <none>           <none>
rook-ceph-osd-prepare-gpu02-brm8v         0/1     Completed   0          2m17s   10.233.112.6     gpu02       <none>           <none>
rook-ceph-rgw-default-a-597466649-8wr6w   1/2     Running     0          10s     192.168.11.103   control03   <none>           <none>
rook-ceph-tools-665997cbf8-v48gx          1/1     Running     0          7m36s   192.168.11.108   gpu02       <none>           <none>
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-rgw --no-headers
++ awk '{print $1; exit}'
+ RGW_POD=rook-ceph-rgw-default-a-597466649-8wr6w
+ [[ -z rook-ceph-rgw-default-a-597466649-8wr6w ]]
+ helm osh wait-for-pods ceph
++ kubectl get pods --namespace=ceph --selector=app=rook-ceph-tools --no-headers
++ grep Running
++ awk '{ print $1; exit }'
+ TOOLS_POD=rook-ceph-tools-665997cbf8-v48gx
+ kubectl exec -n ceph rook-ceph-tools-665997cbf8-v48gx -- ceph -s
  cluster:
    id:     bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 3m) [leader: a]
    mgr: a(active, since 2m), standbys: b, c
    osd: 8 osds: 8 up (since 70s), 8 in (since 86s)
    rgw: 1 daemon active (1 hosts, 1 zones)

  data:
    pools:   10 pools, 121 pgs
    objects: 276 objects, 590 KiB
    usage:   217 MiB used, 14 TiB / 14 TiB avail
    pgs:     121 active+clean

  io:
    client:   127 B/s rd, 127 B/s wr, 0 op/s rd, 0 op/s wr
    recovery: 2 B/s, 0 objects/s
```

결과 확인
```
root@control01:~/osh/openstack-helm# kubectl get pod -A | grep rook
ceph                           rook-ceph-mgr-a-66bbc6dc4b-5rv24                         3/3     Running     0          6m34s
ceph                           rook-ceph-mgr-b-94bd58b6f-lqfwq                          3/3     Running     0          6m34s
ceph                           rook-ceph-mgr-c-9bddbd7b4-qrbz6                          3/3     Running     0          6m34s
ceph                           rook-ceph-mon-a-f5c4df79d-tbb8r                          2/2     Running     0          7m1s
ceph                           rook-ceph-mon-b-795ffcd64d-jvx55                         2/2     Running     0          6m57s
ceph                           rook-ceph-mon-c-7d65b7f68c-nrt9t                         2/2     Running     0          6m47s
ceph                           rook-ceph-osd-0-b8ff77fcd-xf5sk                          2/2     Running     0          4m43s
ceph                           rook-ceph-osd-1-6555ccdbc9-55xws                         2/2     Running     0          4m52s
ceph                           rook-ceph-osd-2-5798984758-xr4vg                         2/2     Running     0          4m43s
ceph                           rook-ceph-osd-3-575ff49f7d-h9km7                         2/2     Running     0          4m52s
ceph                           rook-ceph-osd-4-7c65755ccd-66fcm                         2/2     Running     0          4m43s
ceph                           rook-ceph-osd-5-7d8786698-w6ll9                          2/2     Running     0          4m43s
ceph                           rook-ceph-osd-6-85bfb9cd4c-bvmtb                         2/2     Running     0          4m43s
ceph                           rook-ceph-osd-7-5d78b94bb4-6dbz9                         2/2     Running     0          4m43s
ceph                           rook-ceph-osd-prepare-control02-t8nbh                    0/1     Completed   0          6m10s
ceph                           rook-ceph-osd-prepare-control03-wvldw                    0/1     Completed   0          6m12s
ceph                           rook-ceph-osd-prepare-cpu01-vlxdc                        0/1     Completed   0          6m12s
ceph                           rook-ceph-osd-prepare-cpu02-w2r8s                        0/1     Completed   0          6m11s
ceph                           rook-ceph-osd-prepare-cpu03-gngwt                        0/1     Completed   0          6m11s
ceph                           rook-ceph-osd-prepare-gpu01-vvs46                        0/1     Completed   0          6m10s
ceph                           rook-ceph-osd-prepare-gpu02-brm8v                        0/1     Completed   0          6m10s
ceph                           rook-ceph-rgw-default-a-597466649-8wr6w                  2/2     Running     0          4m3s
ceph                           rook-ceph-tools-665997cbf8-v48gx                         1/1     Running     0          11m
rook-ceph                      ceph-csi-controller-manager-6567d74759-hkw76             1/1     Running     0          12m
rook-ceph                      rook-ceph-operator-bf7b7fc44-m7whm                       1/1     Running     0          12m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-ctrlplugin-79cd9bdf45-7lgjz   5/5     Running     0          10m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-nodeplugin-2qwhh              2/2     Running     0          10m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-nodeplugin-56q2w              2/2     Running     0          10m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-nodeplugin-9m99p              2/2     Running     0          10m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-nodeplugin-dz85n              2/2     Running     0          10m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-nodeplugin-ggb79              2/2     Running     0          10m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-nodeplugin-s86p2              2/2     Running     0          10m
rook-ceph                      rook-ceph.rbd.csi.ceph.com-nodeplugin-v2gs6              2/2     Running     0          10m
```

2.7. ceph 확인
2.7.1. 확인 1
```
root@control01:~/osh/openstack-helm# sed -n '235,245p' tools/deployment/ceph/ceph-rook.sh
    mon: system-node-critical
    osd: system-node-critical
    mgr: system-cluster-critical
  storage: # cluster level storage configuration and selection
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[a-f]"
  disruptionManagement:
    managePodBudgets: true
    osdMaintenanceTimeout: 30
    pgHealthCheckTimeout: 0
root@control01:~/osh/openstack-helm# kubectl -n openstack get sc general -oyaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    meta.helm.sh/release-name: rook-ceph-cluster
    meta.helm.sh/release-namespace: ceph
    storageclass.kubernetes.io/is-default-class: "true"
    storageclass.kubesphere.io/allow-clone: "true"
    storageclass.kubesphere.io/allow-snapshot: "true"
  creationTimestamp: "2026-05-07T01:53:28Z"
  labels:
    app.kubernetes.io/managed-by: Helm
  name: general
  resourceVersion: "2560346"
  uid: e62d295f-1e72-40a8-af8a-86e37b579c0f
parameters:
  clusterID: ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: ceph
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: ceph
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: ceph
  imageFeatures: layering
  imageFormat: "2"
  pool: rbd
provisioner: rook-ceph.rbd.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```
2.7.2. 확인 2
```
root@control01:~/osh/openstack-helm# kubectl get pod -n ceph
NAME                                      READY   STATUS      RESTARTS   AGE
rook-ceph-mgr-a-66bbc6dc4b-5rv24          3/3     Running     0          3h51m
rook-ceph-mgr-b-94bd58b6f-lqfwq           3/3     Running     0          3h51m
rook-ceph-mgr-c-9bddbd7b4-qrbz6           3/3     Running     0          3h51m
rook-ceph-mon-a-f5c4df79d-tbb8r           2/2     Running     0          3h51m
rook-ceph-mon-b-795ffcd64d-jvx55          2/2     Running     0          3h51m
rook-ceph-mon-c-7d65b7f68c-nrt9t          2/2     Running     0          3h51m
rook-ceph-osd-0-b8ff77fcd-xf5sk           2/2     Running     0          3h49m
rook-ceph-osd-1-6555ccdbc9-55xws          2/2     Running     0          3h49m
rook-ceph-osd-2-5798984758-xr4vg          2/2     Running     0          3h49m
rook-ceph-osd-3-575ff49f7d-h9km7          2/2     Running     0          3h49m
rook-ceph-osd-4-7c65755ccd-66fcm          2/2     Running     0          3h49m
rook-ceph-osd-5-7d8786698-w6ll9           2/2     Running     0          3h49m
rook-ceph-osd-6-85bfb9cd4c-bvmtb          2/2     Running     0          3h49m
rook-ceph-osd-7-5d78b94bb4-6dbz9          2/2     Running     0          3h49m
rook-ceph-osd-prepare-control02-t8nbh     0/1     Completed   0          3h50m
rook-ceph-osd-prepare-control03-wvldw     0/1     Completed   0          3h50m
rook-ceph-osd-prepare-cpu01-vlxdc         0/1     Completed   0          3h50m
rook-ceph-osd-prepare-cpu02-w2r8s         0/1     Completed   0          3h50m
rook-ceph-osd-prepare-cpu03-gngwt         0/1     Completed   0          3h50m
rook-ceph-osd-prepare-gpu01-vvs46         0/1     Completed   0          3h50m
rook-ceph-osd-prepare-gpu02-brm8v         0/1     Completed   0          3h50m
rook-ceph-rgw-default-a-597466649-8wr6w   2/2     Running     0          3h48m
rook-ceph-tools-665997cbf8-v48gx          1/1     Running     0          3h55m
```

2.7.3. 확인 3
여기에서 cluster id가 중요하다고 함.
```
root@control01:~/osh/openstack-helm# kubectl exec -it -n ceph rook-ceph-tools-665997cbf8-v48gx  -- /bin/bash
bash-5.1$ ceph status
  cluster:
    id:     bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 3h) [leader: a]
    mgr: a(active, since 3h), standbys: b, c
    osd: 8 osds: 8 up (since 3h), 8 in (since 3h)
    rgw: 1 daemon active (1 hosts, 1 zones)

  data:
    pools:   10 pools, 121 pgs
    objects: 404 objects, 610 KiB
    usage:   218 MiB used, 14 TiB / 14 TiB avail
    pgs:     121 active+clean

  io:
    client:   85 B/s rd, 170 B/s wr, 0 op/s rd, 0 op/s wr
bash-5.1$ ceph osd df
ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA     OMAP     META     AVAIL    %USE  VAR   PGS  STATUS
 0    ssd  1.74660   1.00000  1.7 TiB   27 MiB  796 KiB   25 KiB   26 MiB  1.7 TiB  0.00  1.00   17      up
 2    ssd  1.74660   1.00000  1.7 TiB   27 MiB  848 KiB  208 KiB   26 MiB  1.7 TiB  0.00  1.00   10      up
 4    ssd  1.74660   1.00000  1.7 TiB   27 MiB  824 KiB   24 KiB   26 MiB  1.7 TiB  0.00  1.00   14      up
 5    ssd  1.74660   1.00000  1.7 TiB   27 MiB  820 KiB   25 KiB   26 MiB  1.7 TiB  0.00  1.00   18      up
 6    ssd  1.74660   1.00000  1.7 TiB   27 MiB  828 KiB   27 KiB   26 MiB  1.7 TiB  0.00  1.00   18      up
 7    ssd  1.74660   1.00000  1.7 TiB   28 MiB  1.4 MiB   26 KiB   26 MiB  1.7 TiB  0.00  1.02   17      up
 1    ssd  1.74660   1.00000  1.7 TiB   27 MiB  880 KiB   30 KiB   26 MiB  1.7 TiB  0.00  1.00   19      up
 3    ssd  1.74660   1.00000  1.7 TiB   27 MiB  712 KiB   25 KiB   26 MiB  1.7 TiB  0.00  0.99    8      up
                       TOTAL   14 TiB  218 MiB  7.0 MiB  395 KiB  210 MiB   14 TiB  0.00
MIN/MAX VAR: 0.99/1.02  STDDEV: 0
bash-5.1$ ceph auth ls
osd.0
        key: AQAb8vtp4znLNBAADHNCQ6d2E/xZmK4CkC9u4A==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.1
        key: AQAc8vtp2S/aEBAAOG/NF7jUb1S1LZfru70hkw==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.2
        key: AQAe8vtpCdxQBRAA9N3UwYrrkiGMD9Ru4hdd9Q==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.3
        key: AQAe8vtpX4TvIhAAJR6fll4L8/itdFp/fFZQvQ==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.4
        key: AQAg8vtpIVK/FBAAVyd2pzdeT8SIu53NDtA4TA==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.5
        key: AQAi8vtpBL3pHRAA63si/xatXu45RSYtpa94Ow==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.6
        key: AQAk8vtpEPUALhAAZUfPTlpImfwkZOexlImkpg==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.7
        key: AQAn8vtpFnToARAAqNEA4UyJhCKElby4SiidxQ==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
client.admin
        key: AQDo8PtpzuLuBxAAlijgUoPU2b5tM4jZzWEXRg==
        caps: [mds] allow *
        caps: [mgr] allow *
        caps: [mon] allow *
        caps: [osd] allow *
client.ceph-exporter
        key: AQC88ftp5ftmMBAA7CTmQH9qYD9t2FTKy0XGQg==
        caps: [mds] allow r
        caps: [mgr] allow r
        caps: [mon] allow profile ceph-exporter
        caps: [osd] allow r
client.crash
        key: AQC88ftpxAheIxAAFQaPYErHhdKr9+ZR2rbz+Q==
        caps: [mgr] allow rw
        caps: [mon] allow profile crash
client.csi-cephfs-node.1
        key: AQC88ftpOnVIFRAAvQCFJQtI2iqgAD/Sql1nVA==
        caps: [mds] allow rw
        caps: [mgr] allow rw
        caps: [mon] allow r
        caps: [osd] allow rwx tag cephfs metadata=*, allow rw tag cephfs data=*
client.csi-cephfs-provisioner.1
        key: AQC88ftpbjhCABAA9rQ2G+iv2yD8GTQ0hXlAUg==
        caps: [mds] allow *
        caps: [mgr] allow rw
        caps: [mon] allow r, allow command 'osd blocklist'
        caps: [osd] allow rw tag cephfs metadata=*
client.csi-rbd-node.1
        key: AQC78ftplLyoJhAAQZQ3Qu/wRb0orl78DzF5zw==
        caps: [mgr] allow rw
        caps: [mon] profile rbd
        caps: [osd] profile rbd
client.csi-rbd-provisioner.1
        key: AQC78ftpN5FuERAAt5KAmY5gN+mboBaWNjCswQ==
        caps: [mgr] allow rw
        caps: [mon] profile rbd, allow command 'osd blocklist'
        caps: [osd] profile rbd
client.rbd-mirror-peer
        key: AQC98ftpBqxINRAAVp7GPp+ltrr/++fEpTCoUw==
        caps: [mon] profile rbd-mirror-peer
        caps: [osd] profile rbd
client.rgw.default.a
        key: AQBW8vtpnfigAxAAM1Yo97+9mRpio8d56KsLjg==
        caps: [mon] allow rw
        caps: [osd] allow rwx
mgr.a
        key: AQC/8ftpG/WYABAA86GROwP55H8/Us2Has6Rig==
        caps: [mds] allow *
        caps: [mon] allow profile mgr
        caps: [osd] allow *
mgr.b
        key: AQC/8ftp3/T0DBAAi3UR+OVJ3gQthGzC2QzdAw==
        caps: [mds] allow *
        caps: [mon] allow profile mgr
        caps: [osd] allow *
mgr.c
        key: AQC/8ftpoSdRGRAAljmmbdN/e0dYyQgHZExrbw==
        caps: [mds] allow *
        caps: [mon] allow profile mgr
        caps: [osd] allow *
bash-5.1$ ceph health
HEALTH_OK
bash-5.1$ ceph osd lspools
1 .mgr
2 default.rgw.control
3 rbd
4 default.rgw.meta
5 default.rgw.log
6 default.rgw.buckets.index
7 default.rgw.buckets.non-ec
8 default.rgw.otp
9 .rgw.root
10 default.rgw.buckets.data
```

2.8. 뭔가 했음...
```
kubectl taint nodes control01 node-role.kubernetes.io/control-plane-
```

```
원래는 
확인 방법 : helm get values ceph-adapter-rook -n openstack


오버라이드 밸류를 걍 default value에 입력했음...
확인은 다음과 같이...
root@control01:~/osh/openstack-helm/values_overrides# kubectl get cm -n openstack
NAME                       DATA   AGE
ceph-adapter-rook-bin      2      27m
ceph-etc                   1      27m
ingress-nginx-controller   1      25h
kube-root-ca.crt           1      25h
rabbitmq-rabbitmq-bin      7      4m2s
rabbitmq-rabbitmq-etc      3      4m2s
root@control01:~/osh/openstack-helm/values_overrides# kubectl get cm -n openstack ceph-etc -oyaml
apiVersion: v1
data:
  ceph.conf: |
    [global]
    fsid = bdb3b486-d761-48b6-b3b8-02f9710c1e3a
    mon_host = 192.168.10.102:6789
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"ceph.conf":"[global]\nfsid = bdb3b486-d761-48b6-b3b8-02f9710c1e3a\nmon_host = 192.168.11.107:6789,192.168.11.105:6789,192.168.11.108:6789\n"},"kind":"ConfigMap","metadata":{"annotations":{"meta.helm.sh/release-name":"ceph-adapter-rook","meta.helm.sh/release-namespace":"openstack"},"creationTimestamp":"2026-05-07T06:57:55Z","labels":{"app.kubernetes.io/managed-by":"Helm"},"name":"ceph-etc","namespace":"openstack","resourceVersion":"2646508","uid":"d7acb29c-84c0-455a-95d7-a233c29831b4"}}
    meta.helm.sh/release-name: ceph-adapter-rook
    meta.helm.sh/release-namespace: openstack
  creationTimestamp: "2026-05-07T06:57:55Z"
  labels:
    app.kubernetes.io/managed-by: Helm
  name: ceph-etc
  namespace: openstack
  resourceVersion: "2652205"
  uid: d7acb29c-84c0-455a-95d7-a233c29831b4

-----------------------------------------------------


# fs id만 세팅하고 스크립트 실행
helm upgrade --install ceph-adapter-rook openstack-helm/ceph-adapter-rook --namespace=openstack $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c ceph-adapter-rook ${FEATURES})

ceph-adapter-rook openstack
==>
helm upgrade --install ceph-adapter-rook openstack-helm/ceph-adapter-rook --namespace=openstack \
$(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c ceph-adapter-rook ${FEATURES})

helm upgrade --install ceph-adapter-rook openstack-helm/ceph-adapter-rook --namespace=openstack  \
helm osh get-values-overrides -p /root/osh/openstack-helm/values_overrides -c ceph-adapter-rook 2025.1 ubuntu_noble


archiadmin@k8s-ops-con001-ds2:~/osh/openstack-helm/values_overrides$ vim ceph-adapter-rook/ubuntu_jammy.yaml
conf:
  ceph:
    global:
      fsid: 57503f8e-1523-42ab-994c-0db1635b52ef
      mon_host: ["ip1:6789","ip2:6789","ip3:6789"]
ceph_cluster_namespace: ceph
```

2.9 기타등등 설치
2.9.1. rabbitmq
```
root@control01:~/osh/openstack-helm# helm upgrade --install rabbitmq rabbitmq \
    --namespace=openstack \
    --set pod.replicas.server=3 \
    --timeout=600s \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c rabbitmq ${FEATURES})
Base URL: https://opendev.org/openstack/openstack-helm/raw/branch/master/values_overrides
Base path: /root/osh/openstack-helm/values_overrides
Chart: rabbitmq
Features: 2025.1 ubuntu_noble
Override candidate: /root/osh/openstack-helm/values_overrides/rabbitmq/ubuntu_noble.yaml
File not found: /root/osh/openstack-helm/values_overrides/rabbitmq/ubuntu_noble.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/rabbitmq/2025.1.yaml
File not found: /root/osh/openstack-helm/values_overrides/rabbitmq/2025.1.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/rabbitmq/2025.1-ubuntu_noble.yaml
File found: /root/osh/openstack-helm/values_overrides/rabbitmq/2025.1-ubuntu_noble.yaml
Resulting override args: --values /root/osh/openstack-helm/values_overrides/rabbitmq/2025.1-ubuntu_noble.yaml
Release "rabbitmq" does not exist. Installing it now.
NAME: rabbitmq
LAST DEPLOYED: Thu May  7 16:20:56 2026
NAMESPACE: openstack
STATUS: deployed
REVISION: 1
```
확인
```
root@control01:~/osh/openstack-helm/values_overrides# kubectl get pod -n openstack -o wide
NAME                                                   READY   STATUS      RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
ceph-adapter-rook-namespace-client-ceph-config-4nxsx   0/1     Completed   0          30m     10.233.102.1    control01   <none>           <none>
ceph-adapter-rook-namespace-client-key-xpvjv           0/1     Completed   0          30m     10.233.102.2    control01   <none>           <none>
ingress-nginx-controller-6ccbb554f8-8jr7l              1/1     Running     0          25h     10.233.112.4    gpu02       <none>           <none>
rabbitmq-cluster-wait-764vn                            0/1     Completed   0          7m43s   10.233.102.4    control01   <none>           <none>
rabbitmq-rabbitmq-0                                    1/1     Running     0          7m43s   10.233.102.5    control01   <none>           <none>
rabbitmq-rabbitmq-1                                    1/1     Running     0          7m43s   10.233.102.7    control01   <none>           <none>
rabbitmq-rabbitmq-2                                    1/1     Running     0          7m43s   10.233.102.6    control01   <none>           <none>
```
2.9.2. mariadb
```
root@control01:~/osh/openstack-helm# helm upgrade --install mariadb mariadb \
    --namespace=openstack \
    --set pod.replicas.server=3 \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c mariadb ${FEATURES})
Base URL: https://opendev.org/openstack/openstack-helm/raw/branch/master/values_overrides
Base path: /root/osh/openstack-helm/values_overrides
Chart: mariadb
Features: 2025.1 ubuntu_noble
Override candidate: /root/osh/openstack-helm/values_overrides/mariadb/ubuntu_noble.yaml
File not found: /root/osh/openstack-helm/values_overrides/mariadb/ubuntu_noble.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/mariadb/2025.1.yaml
File not found: /root/osh/openstack-helm/values_overrides/mariadb/2025.1.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/mariadb/2025.1-ubuntu_noble.yaml
File found: /root/osh/openstack-helm/values_overrides/mariadb/2025.1-ubuntu_noble.yaml
Resulting override args: --values /root/osh/openstack-helm/values_overrides/mariadb/2025.1-ubuntu_noble.yaml
Release "mariadb" does not exist. Installing it now.
NAME: mariadb
LAST DEPLOYED: Thu May  7 16:28:16 2026
NAMESPACE: openstack
STATUS: deployed
REVISION: 1
```
확인
```
root@control01:~/osh/openstack-helm/values_overrides# kubectl get pod -n openstack -o wide
NAME                                                   READY   STATUS      RESTARTS        AGE     IP              NODE        NOMINATED NODE   READINESS GATES
ceph-adapter-rook-namespace-client-ceph-config-4nxsx   0/1     Completed   0               33m     10.233.102.1    control01   <none>           <none>
ceph-adapter-rook-namespace-client-key-xpvjv           0/1     Completed   0               33m     10.233.102.2    control01   <none>           <none>
ingress-nginx-controller-6ccbb554f8-8jr7l              1/1     Running     0               25h     10.233.112.4    gpu02       <none>           <none>
mariadb-controller-7f55fd7c67-g249l                    1/1     Running     0               2m58s   10.233.102.8    control01   <none>           <none>
mariadb-server-0                                       1/1     Running     1 (2m19s ago)   2m58s   10.233.102.11   control01   <none>           <none>
mariadb-server-1                                       1/1     Running     0               2m58s   10.233.102.9    control01   <none>           <none>
mariadb-server-2                                       1/1     Running     1 (2m19s ago)   2m58s   10.233.102.10   control01   <none>           <none>
rabbitmq-cluster-wait-764vn                            0/1     Completed   0               10m     10.233.102.4    control01   <none>           <none>
rabbitmq-rabbitmq-0                                    1/1     Running     0               10m     10.233.102.5    control01   <none>           <none>
rabbitmq-rabbitmq-1                                    1/1     Running     0               10m     10.233.102.7    control01   <none>           <none>
rabbitmq-rabbitmq-2                                    1/1     Running     0               10m     10.233.102.6    control01   <none>           <none>
```

2.9.3. memcached
```
root@control01:~/osh/openstack-helm# helm upgrade --install memcached memcached \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c memcached ${FEATURES})
Base URL: https://opendev.org/openstack/openstack-helm/raw/branch/master/values_overrides
Base path: /root/osh/openstack-helm/values_overrides
Chart: memcached
Features: 2025.1 ubuntu_noble
Override candidate: /root/osh/openstack-helm/values_overrides/memcached/ubuntu_noble.yaml
File not found: /root/osh/openstack-helm/values_overrides/memcached/ubuntu_noble.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/memcached/2025.1.yaml
File not found: /root/osh/openstack-helm/values_overrides/memcached/2025.1.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/memcached/2025.1-ubuntu_noble.yaml
File not found: /root/osh/openstack-helm/values_overrides/memcached/2025.1-ubuntu_noble.yaml
Resulting override args:
Release "memcached" does not exist. Installing it now.
NAME: memcached
LAST DEPLOYED: Thu May  7 16:31:46 2026
NAMESPACE: openstack
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
확인
```
root@control01:~/osh/openstack-helm/values_overrides# kubectl get pod -n openstack -o wide
NAME                                                   READY   STATUS      RESTARTS        AGE     IP              NODE        NOMINATED NODE   READINESS GATES
ceph-adapter-rook-namespace-client-ceph-config-4nxsx   0/1     Completed   0               34m     10.233.102.1    control01   <none>           <none>
ceph-adapter-rook-namespace-client-key-xpvjv           0/1     Completed   0               34m     10.233.102.2    control01   <none>           <none>
ingress-nginx-controller-6ccbb554f8-8jr7l              1/1     Running     0               25h     10.233.112.4    gpu02       <none>           <none>
mariadb-controller-7f55fd7c67-g249l                    1/1     Running     0               4m37s   10.233.102.8    control01   <none>           <none>
mariadb-server-0                                       1/1     Running     1 (3m58s ago)   4m37s   10.233.102.11   control01   <none>           <none>
mariadb-server-1                                       1/1     Running     0               4m37s   10.233.102.9    control01   <none>           <none>
mariadb-server-2                                       1/1     Running     1 (3m58s ago)   4m37s   10.233.102.10   control01   <none>           <none>
memcached-memcached-0                                  1/1     Running     0               68s     10.233.102.12   control01   <none>           <none>
rabbitmq-cluster-wait-764vn                            0/1     Completed   0               11m     10.233.102.4    control01   <none>           <none>
rabbitmq-rabbitmq-0                                    1/1     Running     0               11m     10.233.102.5    control01   <none>           <none>
rabbitmq-rabbitmq-1                                    1/1     Running     0               11m     10.233.102.7    control01   <none>           <none>
rabbitmq-rabbitmq-2                                    1/1     Running     0               11m     10.233.102.6    control01   <none>           <none>
```

2.9.4. keystone
```
root@control01:~/osh/openstack-helm# helm upgrade --install keystone keystone \
    --namespace=openstack \
        $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c keystone ${FEATURES})
Base URL: https://opendev.org/openstack/openstack-helm/raw/branch/master/values_overrides
Base path: /root/osh/openstack-helm/values_overrides
Chart: keystone
Features: 2025.1 ubuntu_noble
Override candidate: /root/osh/openstack-helm/values_overrides/keystone/ubuntu_noble.yaml
File not found: /root/osh/openstack-helm/values_overrides/keystone/ubuntu_noble.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/keystone/2025.1.yaml
File not found: /root/osh/openstack-helm/values_overrides/keystone/2025.1.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/keystone/2025.1-ubuntu_noble.yaml
File found: /root/osh/openstack-helm/values_overrides/keystone/2025.1-ubuntu_noble.yaml
Resulting override args: --values /root/osh/openstack-helm/values_overrides/keystone/2025.1-ubuntu_noble.yaml
Release "keystone" does not exist. Installing it now.
NAME: keystone
LAST DEPLOYED: Thu May  7 16:34:05 2026
NAMESPACE: openstack
STATUS: deployed
REVISION: 1
```
확인
```
root@control01:~/osh/openstack-helm/values_overrides# kubectl get pod -n openstack -o wide
NAME                                                   READY   STATUS      RESTARTS       AGE     IP              NODE        NOMINATED NODE   READINESS GATES
ceph-adapter-rook-namespace-client-ceph-config-4nxsx   0/1     Completed   0              39m     10.233.102.1    control01   <none>           <none>
ceph-adapter-rook-namespace-client-key-xpvjv           0/1     Completed   0              39m     10.233.102.2    control01   <none>           <none>
ingress-nginx-controller-6ccbb554f8-8jr7l              1/1     Running     0              25h     10.233.112.4    gpu02       <none>           <none>
keystone-api-6cdcdcfc8d-w45kd                          1/1     Running     0              2m53s   10.233.102.14   control01   <none>           <none>
keystone-bootstrap-qvxkg                               0/1     Completed   0              39s     10.233.102.21   control01   <none>           <none>
keystone-credential-setup-thq6p                        0/1     Completed   0              2m53s   10.233.102.15   control01   <none>           <none>
keystone-db-init-l6cvh                                 0/1     Completed   0              2m22s   10.233.102.16   control01   <none>           <none>
keystone-db-sync-f27n5                                 0/1     Completed   0              2m4s    10.233.102.18   control01   <none>           <none>
keystone-domain-manage-qnx4n                           0/1     Completed   0              94s     10.233.102.20   control01   <none>           <none>
keystone-fernet-setup-xp88k                            0/1     Completed   0              2m15s   10.233.102.17   control01   <none>           <none>
keystone-rabbit-init-kjsnf                             0/1     Completed   0              108s    10.233.102.19   control01   <none>           <none>
mariadb-controller-7f55fd7c67-g249l                    1/1     Running     0              8m44s   10.233.102.8    control01   <none>           <none>
mariadb-server-0                                       1/1     Running     1 (8m5s ago)   8m44s   10.233.102.11   control01   <none>           <none>
mariadb-server-1                                       1/1     Running     0              8m44s   10.233.102.9    control01   <none>           <none>
mariadb-server-2                                       1/1     Running     1 (8m5s ago)   8m44s   10.233.102.10   control01   <none>           <none>
memcached-memcached-0                                  1/1     Running     0              5m15s   10.233.102.12   control01   <none>           <none>
rabbitmq-cluster-wait-764vn                            0/1     Completed   0              16m     10.233.102.4    control01   <none>           <none>
rabbitmq-rabbitmq-0                                    1/1     Running     0              16m     10.233.102.5    control01   <none>           <none>
rabbitmq-rabbitmq-1                                    1/1     Running     0              16m     10.233.102.7    control01   <none>           <none>
rabbitmq-rabbitmq-2                                    1/1     Running     0              16m     10.233.102.6    control01   <none>           <none>
```
2.9.5. heat
```
root@control01:~/osh/openstack-helm# helm upgrade --install heat heat \
    --namespace=openstack \
        $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c heat ${FEATURES})
Base URL: https://opendev.org/openstack/openstack-helm/raw/branch/master/values_overrides
Base path: /root/osh/openstack-helm/values_overrides
Chart: heat
Features: 2025.1 ubuntu_noble
Override candidate: /root/osh/openstack-helm/values_overrides/heat/ubuntu_noble.yaml
File not found: /root/osh/openstack-helm/values_overrides/heat/ubuntu_noble.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/heat/2025.1.yaml
File not found: /root/osh/openstack-helm/values_overrides/heat/2025.1.yaml
Override candidate: /root/osh/openstack-helm/values_overrides/heat/2025.1-ubuntu_noble.yaml
File found: /root/osh/openstack-helm/values_overrides/heat/2025.1-ubuntu_noble.yaml
Resulting override args: --values /root/osh/openstack-helm/values_overrides/heat/2025.1-ubuntu_noble.yaml
Release "heat" does not exist. Installing it now.
NAME: heat
LAST DEPLOYED: Thu May  7 16:38:37 2026
NAMESPACE: openstack
STATUS: deployed
REVISION: 1
```
확인
```
root@control01:~/osh/openstack-helm/values_overrides# kubectl get pod -n openstack -o wide
NAME                                                   READY   STATUS      RESTARTS      AGE     IP              NODE        NOMINATED NODE   READINESS GATES
ceph-adapter-rook-namespace-client-ceph-config-4nxsx   0/1     Completed   0             45m     10.233.102.1    control01   <none>           <none>
ceph-adapter-rook-namespace-client-key-xpvjv           0/1     Completed   0             45m     10.233.102.2    control01   <none>           <none>
heat-api-fb9f7d775-nx5dt                               1/1     Running     0             4m33s   10.233.102.24   control01   <none>           <none>
heat-bootstrap-8xjb2                                   0/1     Completed   0             2m39s   10.233.102.32   control01   <none>           <none>
heat-cfn-5f545754f7-c97j6                              1/1     Running     0             4m33s   10.233.102.23   control01   <none>           <none>
heat-db-init-bs69c                                     0/1     Completed   0             4m33s   10.233.102.25   control01   <none>           <none>
heat-db-sync-nn84z                                     0/1     Completed   0             4m25s   10.233.102.26   control01   <none>           <none>
heat-domain-ks-user-stw8m                              0/1     Completed   0             2m28s   10.233.102.33   control01   <none>           <none>
heat-engine-75ccbc98f4-5l7lh                           1/1     Running     0             4m33s   10.233.102.22   control01   <none>           <none>
heat-engine-cleaner-29635660-b72m5                     0/1     Completed   0             3m14s   10.233.102.31   control01   <none>           <none>
heat-ks-endpoints-tlx8w                                0/6     Completed   0             3m41s   10.233.102.29   control01   <none>           <none>
heat-ks-service-7nphw                                  0/2     Completed   0             3m53s   10.233.102.28   control01   <none>           <none>
heat-ks-user-z2g2k                                     0/2     Completed   0             3m19s   10.233.102.30   control01   <none>           <none>
heat-rabbit-init-9z47r                                 0/1     Completed   0             3m59s   10.233.102.27   control01   <none>           <none>
heat-trusts-ngzj7                                      0/1     Completed   0             109s    10.233.102.34   control01   <none>           <none>
ingress-nginx-controller-6ccbb554f8-8jr7l              1/1     Running     0             25h     10.233.112.4    gpu02       <none>           <none>
keystone-api-6cdcdcfc8d-w45kd                          1/1     Running     0             9m5s    10.233.102.14   control01   <none>           <none>
keystone-bootstrap-qvxkg                               0/1     Completed   0             6m51s   10.233.102.21   control01   <none>           <none>
keystone-credential-setup-thq6p                        0/1     Completed   0             9m5s    10.233.102.15   control01   <none>           <none>
keystone-db-init-l6cvh                                 0/1     Completed   0             8m34s   10.233.102.16   control01   <none>           <none>
keystone-db-sync-f27n5                                 0/1     Completed   0             8m16s   10.233.102.18   control01   <none>           <none>
keystone-domain-manage-qnx4n                           0/1     Completed   0             7m46s   10.233.102.20   control01   <none>           <none>
keystone-fernet-setup-xp88k                            0/1     Completed   0             8m27s   10.233.102.17   control01   <none>           <none>
keystone-rabbit-init-kjsnf                             0/1     Completed   0             8m      10.233.102.19   control01   <none>           <none>
mariadb-controller-7f55fd7c67-g249l                    1/1     Running     0             14m     10.233.102.8    control01   <none>           <none>
mariadb-server-0                                       1/1     Running     1 (14m ago)   14m     10.233.102.11   control01   <none>           <none>
mariadb-server-1                                       1/1     Running     0             14m     10.233.102.9    control01   <none>           <none>
mariadb-server-2                                       1/1     Running     1 (14m ago)   14m     10.233.102.10   control01   <none>           <none>
memcached-memcached-0                                  1/1     Running     0             11m     10.233.102.12   control01   <none>           <none>
rabbitmq-cluster-wait-764vn                            0/1     Completed   0             22m     10.233.102.4    control01   <none>           <none>
rabbitmq-rabbitmq-0                                    1/1     Running     0             22m     10.233.102.5    control01   <none>           <none>
rabbitmq-rabbitmq-1                                    1/1     Running     0             22m     10.233.102.7    control01   <none>           <none>
rabbitmq-rabbitmq-2                                    1/1     Running     0             22m     10.233.102.6    control01   <none>           <none>
```

2.9.6. glance
```
root@control01:~/osh/openstack-helm/values_overrides# kubectl get pod -A | grep ceph-tools
ceph                           rook-ceph-tools-665997cbf8-v48gx                         1/1     Running     0             5h50m
root@control01:~/osh/openstack-helm/values_overrides# kubectl exec -it -n ceph rook-ceph-tools-665997cbf8-v48gx  -- /bin/bash
bash-5.1$ ceph auth get-key client.admin
AQDo8PtpzuLuBxAAlijgUoPU2b5tM4jZzWEXRg==bash-5.1$
```
key = "AQDo8PtpzuLuBxAAlijgUoPU2b5tM4jZzWEXRg=="

```
다음과 같이 수정

root@control01:~/osh/openstack-helm/glance# sed -n '120,130p' values.yaml
            failure_rate:
              max: 0
  ceph:
    monitors: []
    enabled: true
    admin_keyring: AQDo8PtpzuLuBxAAlijgUoPU2b5tM4jZzWEXRg==
    override:
    append:
  ceph_client:
    override:
    append:
root@control01:~/osh/openstack-helm/glance# sed -n '270,280p' values.yaml
      service_type: image
    glance_store:
      # Since 2024.1 this section must contain the only key 'default_backend'.
      # Other keys should be defined in the corresponding per-backend sections.
      # This is for backward compatibility.
      default_backend: rbd
      filesystem_store_datadir: /var/lib/glance/images
      cinder_catalog_info: volumev3::internalURL
      rbd_store_chunk_size: 8
      rbd_store_replication: 3
      rbd_store_crush_rule: replicated_rule
```
