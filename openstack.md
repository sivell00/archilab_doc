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
