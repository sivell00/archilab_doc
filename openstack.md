# 1. 사전설정
```
# root 사용자 password 설정 및 root login 활성화
passwd
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config

# sds 계정에 sudo 설정
sed -i '$a sds ALL=(ALL) NOPASSWD: ALL' /etc/sudoers
usermod -a -G sudo sds
```

# 2. Package 설치
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
