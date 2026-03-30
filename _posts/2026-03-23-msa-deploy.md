---
layout: post
title: MSA Deploy
date: 2026-03-23 21:00:00 + 0900
categories: [kubernetes]
tags: [kubernetes, msa]
mermaid: true
---

## 가상머신 환경
마스터노드 > m-k8s, 127.0.0.1:60010, CPU:2, memomry:3072
워커노드 #1 > w1-k8s 127.0.0.1:60101, CPU:1, memory:2560
워커노드 #2 > w2-k8s 127.0.0.1:60102, CPU:1, memory:2560
워커노드 #3 > w3-k8s 127.0.0.1:60103, CPU:1, memory:2560

프로비저닝 도구: vagrant 
하이퍼바이저: virtualbox
가상머신: centos

하이퍼바이저: 하나의 물리 서버에서 CPU, 메모리, 디스크 같은 물리 자원을 나눠서 여러 개의 가상 머신(VM)을 동시에 실행할 수 있게 해주는 소프트웨어. 각각의 VM은 독립된 컴퓨터처럼 동작하고 서로 영향을 주지 않음

```
// vargrantfile
// install_pkg.sh VM에 필요한 패키지 설치를 위한 쉘스크립트

# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  N = 3 # max number of worker nodes
  Ver = '1.18.4' # Kubernetes Version to install
  
  #=============#
  # Master Node #
  #=============#

    config.vm.define "m-k8s" do |cfg|
      cfg.vm.box = "sysnet4admin/CentOS-k8s"
      cfg.vm.provider "virtualbox" do |vb|
        vb.name = "m-k8s(github_SysNet4Admin)"
        vb.cpus = 2
        vb.memory = 3072
        vb.customize ["modifyvm", :id, "--groups", "/k8s-SgMST-18.9.9(github_SysNet4Admin)"]
      end
      cfg.vm.host_name = "m-k8s"
      cfg.vm.network "private_network", ip: "192.168.1.10"
      cfg.vm.network "forwarded_port", guest: 22, host: 60010, auto_correct: true, id: "ssh"
      cfg.vm.synced_folder "../data", "/vagrant", disabled: true 
      cfg.vm.provision "shell", path: "config.sh", args: N
      cfg.vm.provision "shell", path: "install_pkg.sh", args: [ Ver, "Main" ]
      cfg.vm.provision "shell", path: "master_node.sh"
    end

  #==============#
  # Worker Nodes #
  #==============#

  (1..N).each do |i|
    config.vm.define "w#{i}-k8s" do |cfg|
      cfg.vm.box = "sysnet4admin/CentOS-k8s"
      cfg.vm.provider "virtualbox" do |vb|
        vb.name = "w#{i}-k8s(github_SysNet4Admin)"
        vb.cpus = 1
        vb.memory = 2560
        vb.customize ["modifyvm", :id, "--groups", "/k8s-SgMST-18.9.9(github_SysNet4Admin)"]
      end
      cfg.vm.host_name = "w#{i}-k8s"
      cfg.vm.network "private_network", ip: "192.168.1.10#{i}"
      cfg.vm.network "forwarded_port", guest: 22, host: "6010#{i}", auto_correct: true, id: "ssh"
      cfg.vm.synced_folder "../data", "/vagrant", disabled: true
      cfg.vm.provision "shell", path: "config.sh", args: N
      cfg.vm.provision "shell", path: "install_pkg.sh", args: Ver
      cfg.vm.provision "shell", path: "work_nodes.sh"
    end
  end

end
```

## 쿠버네티스 환경
쿠버네티스 클러스터 구성 솔루션: kubeadm
config.sh에 kubeadm으로 쿠버네티스를 설치하기 위한 사전 조건을 작성
master_node.sh는 m-k8s 가상머신을 쿠버네티스 마스터 노드와 컨테이너 네트워크 인터페이스를 구성하는 스크립트
work_nodes.sh는 w1,2,3-k8s 3대의 가상머신에 쿠버네티스 워커 노드를 구성하는 스크립트

```bash
// config.sh 

#!/usr/bin/env bash

# vim configuration 
echo 'alias vi=vim' >> /etc/profile

# swapoff -a to disable swapping
swapoff -a
# sed to comment the swap partition in /etc/fstab
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

# CentOS repo change from mirror to vault 
sed -i -e 's/mirrorlist=/#mirrorlist=/g' /etc/yum.repos.d/CentOS-*
sed -i -e 's/mirrorlist=/#mirrorlist=/g' /etc/yum.conf
sed -E -i -e 's/#baseurl=http:\/\/mirror.centos.org\/centos\/\$releasever\/([[:alnum:]_-]*)\/\$basearch\//baseurl=https:\/\/vault.centos.org\/7.9.2009\/\1\/\$basearch\//g' /etc/yum.repos.d/CentOS-*
sed -E -i -e 's/#baseurl=http:\/\/mirror.centos.org\/centos\/\$releasever\/([[:alnum:]_-]*)\/\$basearch\//baseurl=https:\/\/vault.centos.org\/7.9.2009\/\1\/\$basearch\//g' /etc/yum.conf

# kubernetes repo
gg_pkg="http://mirrors.aliyun.com/kubernetes/yum" # Due to shorten addr for key
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=${gg_pkg}/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=${gg_pkg}/doc/yum-key.gpg ${gg_pkg}/doc/rpm-package-key.gpg
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# RHEL/CentOS 7 have reported traffic issues being routed incorrectly due to iptables bypassed
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
modprobe br_netfilter

# local small dns & vagrant cannot parse and delivery shell code.
echo "192.168.1.10 m-k8s" >> /etc/hosts
for (( i=1; i<=$1; i++  )); do echo "192.168.1.10$i w$i-k8s" >> /etc/hosts; done

# config DNS  
cat <<EOF > /etc/resolv.conf
nameserver 1.1.1.1 #cloudflare DNS
nameserver 8.8.8.8 #Google DNS
EOF

# docker repo
yum install yum-utils -y 
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

```bash
// master_node.sh

#!/usr/bin/env bash

# init kubernetes 
kubeadm init --token 123456.1234567890123456 --token-ttl 0 \
--pod-network-cidr=172.16.0.0/16 --apiserver-advertise-address=192.168.1.10 

# config for master node only 
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# config for kubernetes's network 
kubectl apply -f \
https://raw.githubusercontent.com/sysnet4admin/IaC/master/manifests/172.16_net_calico.yaml
```

```bash
// work_nodes.sh

#!/usr/bin/env bash

# config for work_nodes only 
kubeadm join --token 123456.1234567890123456 \
             --discovery-token-unsafe-skip-ca-verification 192.168.1.10:6443
```

## Repositories
1. 구성
w1-k8s: kafka
w2-k8s: MySql master
w3-k8s: MySql slave + Redis

2. MySql
2-1. namespace: deploy-test-data

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: deploy-test-data
```

2-2. PV / PVC
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-master-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/mysql-master
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - w2-k8s
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-slave-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/mysql-slave
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - w3-k8s
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-master-pvc
  namespace: deploy-test-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeName: mysql-master-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-slave-pvc
  namespace: deploy-test-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeName: mysql-slave-pv
```

2-2. Headless Service
master
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
  namespace: deploy-test-data
spec:
  clusterIP: None
  selector:
    app: mysql-master
  ports:
    - port: 3306
```

slave
```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: mysql-slave
    namespace: deploy-test-data
  spec:
    clusterIP: None
    selector:
      app: mysql-slave
    ports:
      - port: 3306
```

2-3. ConfigMap
master
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-master-config
  namespace: deploy-test-data
data:
  master.cnf: |
    [mysqld]
    server-id=1
    log-bin=mysql-bin
    binlog_format=ROW

    gtid_mode=ON
    enforce_gtid_consistency=ON
    log_slave_updates=ON
```

slave
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-slave-config
  namespace: deploy-test-data
data:
  slave.cnf: |
    [mysqld]
    server-id=2
    relay-log=mysql-relay-bin

    gtid_mode=ON
    enforce_gtid_consistency=ON
    log_slave_updates=ON
```

2-4. StatefulSet
master
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-master
  namespace: deploy-test-data
spec:
  serviceName: mysql-master
  replicas: 1

  selector:
    matchLabels:
      app: mysql-master

  template:
    metadata:
      labels:
        app: mysql-master
    spec:
      nodeSelector:
        kubernetes.io/hostname: w2-k8s

      containers:
      - name: mysql
        image: mysql:8.0

        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpass

        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d

        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"

      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: mysql-master-pvc
      - name: config
        configMap:
          name: mysql-master-config
```

slave
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-slave
  namespace: deploy-test-data
spec:
  serviceName: mysql-slave
  replicas: 1

  selector:
    matchLabels:
      app: mysql-slave

  template:
    metadata:
      labels:
        app: mysql-slave
    spec:
      nodeSelector:
        kubernetes.io/hostname: w3-k8s

      containers:
      - name: mysql
        image: mysql:8.0

        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpass

        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d

        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"

      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: mysql-slave-pvc
      - name: config
        configMap:
          name: mysql-slave-config
```

2-5. MySql 접근 설정
m-k8s에서 아래와 같이 포트포워딩을 실행 root로 접속
```bash
kubectl port-forward -n deploy-test-data pod/mysql-master-0 13306:3306
kubectl port-forward -n deploy-test-data pod/mysql-slave-0 23306:3306
```

그 다음 로컬 PC의 터미널에서 아래 실행하고 workbench에서 Host: 127.0.0.1, Port: 13306(또는 23306), User: root
```bash
ssh -p 60010 -L 13306:127.0.0.1:13306 root@127.0.0.1  // master
ssh -p 60010 -L 23306:127.0.0.1:23306 root@127.0.0.1  // slave
```

2-6. Replication 설정
replication 전용 사용자를 만든다. mysql-master에 접속해 아래 명령어를 실행한다.

```sql
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

Slave가 Master의 데이터를 복제하도록 설정하는 명령어
GTID 기반 복제를 사용한다. GTID는 각 트랜잭션에 고유 ID를 부여해 Slave는 Master에서 어디까지 복제했는지 기억하고 이어서 복제한다.

```sql
CHANGE MASTER TO
  MASTER_HOST='mysql-master-0.mysql-master',
  MASTER_USER='repl',
  MASTER_PASSWORD='replpass',
  MASTER_AUTO_POSITION=1;               // GTID 기반 복제 사용

START SLAVE;                            // 실제로 복제를 시작하는 명령
```

위 명령어가 ```CHANGE REPLICATION SOURCE TO SOURCE_AUTO_POSITION = 1 cannot be executed because @@GLOBAL.GTID_MODE = OFF.``` 에러로 실패하면 MySQL에 GTID가 꺼져 있어서 실패한 상황이다.
아래 Slave MySQL에서 명령어로 GTID를 동적으로 켜준다. MySQL 서버가 재기동 되면 설정이 유지되지 않을 수 있다.

```sql
SET GLOBAL enforce_gtid_consistency = ON;
SET GLOBAL gtid_mode = OFF_PERMISSIVE;
SET GLOBAL gtid_mode = ON_PERMISSIVE;
SET GLOBAL gtid_mode = ON;
```

Master, Slave 동작은 아래 쿼리를 실행하고 Slave_IO_Running, Slave_SQL_Running이 YES인지 확인한다.

```sql
SHOW SLAVE STATUS;
```

실행결과
<figure>
  <img src="https://i.imgur.com/oUEgSQI.jpeg" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">SHOW SLAVE STATUS 결과</p>
</figure>

## MSA 환경
1. 네임스페이스: deploy-test
kubectl create namespace deploy-test

2. deployment

3. service

4. ingress

## devOps 환경
1. jenkins

2. hobor

3. argocd

## 모니터링
1. grafana

2. lgtm

3. slack