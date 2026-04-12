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
로컬 PC의 터미널에서 아래 실행
```bash
ssh -p 60010 -L 13306:127.0.0.1:13306 root@127.0.0.1  // master
ssh -p 60010 -L 23306:127.0.0.1:23306 root@127.0.0.1  // slave
```

그 다음 m-k8s에서 아래와 같이 포트포워딩을 실행 root로 접속하고 workbench에서 Host: 127.0.0.1, Port: 13306(또는 23306), User: root으로 접속
```bash
kubectl port-forward -n deploy-test-data pod/mysql-master-0 13306:3306
kubectl port-forward -n deploy-test-data pod/mysql-slave-0 23306:3306
```

※ ssh 접속과 포트포워딩 순서를 반대로 하면 m-k8s 노드로 ssh 접속이 안될 수 있음

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

주문 서비스를 위한 database를 생성한다.
```sql
create database ordermsa;
```

3. Kafka
```yaml
# Zookeeper Deployment + Service

apiVersion: apps/v1
kind: Deployment
metadata:
  name: zookeeper
  namespace: deploy-test-data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      nodeSelector:
        kubernetes.io/hostname: w1-k8s
      containers:
      - name: zookeeper
        image: wurstmeister/zookeeper
        ports:
        - containerPort: 2181
---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper-service
  namespace: deploy-test-data
spec:
  type: ClusterIP
  ports:
  - port: 2181
    targetPort: 2181
  selector:
    app: zookeeper
```

```yaml
# Kafka Deployment + Service

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka
  namespace: deploy-test-data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      nodeSelector:
        kubernetes.io/hostname: w1-k8s
      containers:
      - name: kafka
        image: wurstmeister/kafka
        ports:
        - containerPort: 9092
        - containerPort: 9093
        env:
        - name: KAFKA_BROKER_ID
          value: "1"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zookeeper-service:2181"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "INSIDE://kafka-service.deploy-test-data.svc.cluster.local:9093,OUTSIDE://kafka-service.deploy-test-data.svc.cluster.local:9092"
        - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
          value: "INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT"
        - name: KAFKA_LISTENERS
          value: "INSIDE://0.0.0.0:9093,OUTSIDE://0.0.0.0:9092"
        - name: KAFKA_INTER_BROKER_LISTENER_NAME
          value: "INSIDE"

---
apiVersion: v1
kind: Service
metadata:
  name: kafka-service
  namespace: deploy-test-data
spec:
  type: ClusterIP
  ports:
  - name: outside
    port: 9092
    targetPort: 9092
  - name: inside
    port: 9093
    targetPort: 9093
  selector:
    app: kafka
```

4. Redis

```yaml
iapiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: deploy-test-data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      nodeSelector:
        kubernetes.io/hostname: w3-k8s
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379

        args:
        - "--appendonly"
        - "yes"

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: deploy-test-data
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
```

## MSA 환경
1. 네임스페이스: deploy-test
```
kubectl create namespace deploy-test
```

2. deployment/service
2-1. 회원 서비스
- 로컬에서 이미지를 빌드한 이미지를 k8s에 직접 실행. CI/CD 환경은 추후 구성 예정
```bash
# docker desktop 실행 후 로컬에서 이미지 빌드
docker build -t member-service:latest .

# tar 파일로 저장
docker save -o member-service.tar member-service:latest

# w1-k8s, w2-k8s, w3-k8s로 전송
scp -P 60101 member-service.tar root@127.0.0.1:/root/
scp -P 60102 member-service.tar root@127.0.0.1:/root/
scp -P 60103 member-service.tar root@127.0.0.1:/root/

# w1-k8s, w2-k8s, w3-k8s 각 노드에서 아래 명령어를 실행해 Docker에 이미지 등록
docker load -i member-service.tar
```

- deployment, service 배포
```bash
# m-k8s에서 다음 명령어로 아래 depl_svc.yaml을 실행
kubectl apply -f deply_svc.yaml

# 확인
kubectl get pod -n deploy-test -o wide
kubectl get svc -n deploy-test
```

```yaml
# depl_svc.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: member-depl
  namespace: deploy-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: member
  template:
    metadata:
      labels:
        app: member
    spec:
      containers:
      - name: member-container
        image: member-service:latest
        imagePullPolicy: Never

        ports:
        - containerPort: 8080

        resources:
          limits:
            cpu: "250m"
            memory: "500Mi"
          requests:
            cpu: "100m"
            memory: "250Mi"

        env:
        - name: DB_HOST
          value: "mysql-master-0.mysql-master.deploy-test-data.svc.cluster.local"
        - name: DB_PW
          value: "rootpass"

        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: member-service
  namespace: deploy-test
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: member
```

3. 외부 연결
EKS에서는 LB가 자동으로 붙지만, 온프레미스 + VirtualBox + LB 없는 환경에서는 MetalLB + Ingress 조합을 사용해 외부의 요청을 처리할 수 있다. 

3-1. 전체 구조
```
외부 요청
   ↓
[ MetalLB (외부 IP 할당) ]
   ↓
[ Ingress Controller (Nginx) ]
   ↓
[ member-service (ClusterIP) ]
   ↓
[ member Pod ]
```

3-2. 전체 진행 순서
1️⃣ MetalLB 설치
2️⃣ IP Pool 설정
3️⃣ Ingress Controller 설치
4️⃣ Ingress 리소스 생성
5️⃣ hosts 설정
6️⃣ 접속 테스트

3-3. MetalLB 설치

```bash
# ✔️ v0.11.x 사용
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml

# 설치 후 확인
kubectl get pods -n metallb-system
```

v0.11은 ConfigMap 방식으로 설정해야함. 아래 config를 적용한다

```bash
kubectl apply -f metallb-config.yaml
```

```yaml
# metalLB-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.200-192.168.1.210
```

3-4. Ingress Controller 설치(Nginx)
```bash
# 구버전 ingress-nginx (k8s 1.17~1.18 호환)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/cloud/deploy.yaml

# 설치 확인
kubectl get svc -n ingress-nginx
kubectl get pods -n ingress-nginx
```

3-5. Ingress 리소스 생성

```bash
kubectl apply -f ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: order-backend-ingress
  namespace: deploy-test
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - host: server.deploy-test.shop
    http:
      paths:
      - path: /member-service(/|$)(.*)
        backend:
          serviceName: member-service
          servicePort: 80

      - path: /ordering-service(/|$)(.*)
        backend:
          serviceName: ordering-service
          servicePort: 80

      - path: /product-service(/|$)(.*)
        backend:
          serviceName: product-service
          servicePort: 80
```

```bash
# 적용 후 확인
kubectl get ingress -n deploy-test
kubectl get svc -n ingress-nginx
```

3-6. 도메인 설정
로컬 PC에 host 설정
이 설정으로 ```server.deploy-test.shop```으로 들어온 요청은 실제 DNS에 등록이 안되어 있어 host 파일을 통해 ingress-nginx-controller의 EXTERNAL-IP인 192.168.x.x IP로 강제로 알려줌
```
C:\Windows\System32\drivers\etc\hosts 에서
192.168.1.202 server.deploy-test.shop 추가
192.168.x.x의 주소는 kubectl get svc -n ingress-nginx 실행 결과 LoadBalancer 타임의 EXTERNAL-IP를 참고
```

3-7. 연결 확인
postman으로 아래와 같이 ```POST http://server.deploy-test.shop/member-service/member/doLogin``` 요청 시 응답 확인

<figure>
  <img src="https://i.imgur.com/jkXHovD.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">member-service API 요청 결과</p>
</figure>

3-8. 연결 주의사항

아래와 같이 노드 IP 대역과 LoabBalancer(EXTERNAL-IP) 대역이 다르면 로컬 PC에서 보낸 API 요청이 노드에 전송되지 않는다.   

```bash
[root@m-k8s metalLB]# kubectl get nodes -o wide
NAME     STATUS   ROLES    AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
m-k8s    Ready    master   313d   v1.18.4   192.168.1.10    <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   docker://18.9.9
w1-k8s   Ready    <none>   313d   v1.18.4   192.168.1.101   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   docker://18.9.9
w2-k8s   Ready    <none>   313d   v1.18.4   192.168.1.102   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   docker://18.9.9
w3-k8s   Ready    <none>   313d   v1.18.4   192.168.1.103   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   docker://18.9.9

[root@m-k8s ordering-service]# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.109.70.19     192.168.56.202   80:30701/TCP,443:31605/TCP   5d21h
ingress-nginx-controller-admission   ClusterIP      10.101.148.210   <none>           443/TCP                      5d21h
```

L2(ARP) 레벨에서 패킷이 전달되지 않기 때문이다.   
MetalLB는 L2(ARP) 방식이고 MetalLB는 IP를 가지는 장비가 아니라 노드가 그 IP를 대신 응답하도록 만들어주는 역할이다.   
그리고 ARP는 같은 네트워크에서 IP → MAC 주소를 찾는 프로토콜이다. 이건 같은 네트워크(브로드캐스트 도메인)에서만 가능하다.   
MetalLB speaker는 자기 노드의 네트워크 인터페이스에서만 ARP 응답을 할 수 있다.   
위와 같이 노드 IP와 MetalLB External IP가 다르면 PC가 보낸 192.168.56.202 요청에 아무도 응답하지 않는다.   
실제 구조는 아래와 같다.   

```
// MetalLB 동작 방식
[내 PC]
   ↓
[192.168.56.202 ← 가상의 IP]
   ↓
[Node 중 하나가 대신 응답]
   ↓
[ingress-nginx Pod]
```

참고로 AWS Load Balancer와 같은 보편적인 LB는 실제 서비스/장비이고 트래픽을 MetalLB처럼 노드가 대신 받지 않고 LB가 직접 받는다.   
네트워크 방식은 L3/L4/L7 라우팅을 사용하고 노드와 동일한 네트워크일 필요가 없다.   

```
// AWS Load Balancer 동작 방식
[내 PC]
   ↓
(3.35.x.x ← 진짜 AWS LB IP)
   ↓
[AWS Load Balancer (실제 존재)]
   ↓
[EC2 / Pod]
```

이전 MetalLB 설정은 아래와 같이 ```192.168.56.200-192.168.56.210```으로 되어있어 External IP 잘못 설정 되었다.

```yaml
# metalLB-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.56.200-192.168.56.210
```

아래와 같이 수정하고 명령어를 순서대로 실행해 적용한다.

```yaml
# metalLB-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.200-192.168.1.210
```

```bash
kubectl delete pod -n metallb-system -l app=metallb
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/cloud/deploy.yaml
kubectl get svc -n ingress-nginx
```

그러면 아래와 같이 노드와 같은 대역으로 External IP가 설정되고 로컬 PC에서 보내는 API 요청을 받을 수 있다.    

```bash
[root@m-k8s ~]# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.105.118.119   192.168.1.202   80:31872/TCP,443:32480/TCP   4s
ingress-nginx-controller-admission   ClusterIP      10.101.148.210   <none>          443/TCP                      5d22h
```

4. 그 외 deployment/service

주문, 제품 서비스의 빌드와 deployment, service 설치 방법은 회원 서비스와 동일하다.

4-1. 주문 서비스

```yaml
# depl_svc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ordering-depl
  namespace: deploy-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ordering
  template:
    metadata:
      labels:
        app: ordering
    spec:
      containers:
      - name: ordering-container
        image: ordering-service:latest
        imagePullPolicy: Never

        ports:
        - containerPort: 8080
        
        resources:
        # 컨테이너가 사용할수 있는 리소스의 최대치
          limits:
            cpu: "250m"
            memory: "500Mi"
        # 컨테이너가 시작될떄 보장받아야 하는 최소 자원
          requests:
            cpu: "100m"
            memory: "250Mi"
        env:
        - name: DB_HOST
          value: "mysql-master-0.mysql-master.deploy-test-data.svc.cluster.local"
        - name: DB_PW
          value: "rootpass"
        
        # 컨테이너 상태 확인 
        readinessProbe:
          httpGet:
            # healthcheck 경로
            path: /health
            port: 8080
          # 컨테이너 시작 후 지연
          initialDelaySeconds: 10
          # 확인 반복 주기
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: ordering-service
  namespace: deploy-test
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: ordering

```

4-2. 제품 서비스

```yaml
# depl_svc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-depl
  namespace: deploy-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product
  template:
    metadata:
      labels:
        app: product
    spec:
      containers:
      - name: product-container
        image: product-service:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8080
        resources:
        # 컨테이너가 사용할수 있는 리소스의 최대치
          limits:
            cpu: "250m"
            memory: "500Mi"
        # 컨테이너가 시작될떄 보장받아야 하는 최소 자원
          requests:
            cpu: "100m"
            memory: "250Mi"
        env:
        - name: DB_HOST
          value: "mysql-master-0.mysql-master.deploy-test-data.svc.cluster.local"
        - name: DB_PW
          value: "rootpass"
        # 컨테이너 상태 확인 
        readinessProbe:
          httpGet:
            # healthcheck 경로
            path: /health
            port: 8080
          # 컨테이너 시작 후 지연
          initialDelaySeconds: 10
          # 확인 반복 주기
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: deploy-test
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: product

```


5. 멤버, 제품, 주문 서비스 실행 결과

5-1. 멤버 서비스

로그인
<figure>
  <img src="https://i.imgur.com/ZAoFnmv.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">로그인</p>
</figure>

5-2. 제품 서비스

제품 등록
<figure>
  <img src="https://i.imgur.com/8m7U05h.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">제품 등록 API</p>
</figure>

<figure>
  <img src="https://i.imgur.com/ZBnm4pE.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">제품 DB 등록</p>
</figure>

5-3. 주문 서비스

제품 주문 후 수량 확인   
제품을 주문하면 Kafka의 update-stock-topic 토픽에 주문 메시지를 pub하고 제품 서비스에서 sub 해서 주문 수량 만큼 재고를 줄인다.   

<figure>
  <img src="https://i.imgur.com/MH74JtA.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">제품 주문</p>
</figure>

<figure>
  <img src="https://i.imgur.com/WRiWPO1.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">제품 주문 후 수량</p>
</figure>

## devOps 환경
1. jenkins

2. hobor

3. argocd

## 모니터링
1. grafana

2. lgtm

3. slack

## 그 외
1. 시크릿 설정

2. HTTPS 통신

3. DB 이중화 연결

4. 서비스용 DB 계정 생성

5. 젠킨스, harbor 구성

6. HPA