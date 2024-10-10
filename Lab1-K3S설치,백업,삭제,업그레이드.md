## Lab1. K3S 설치, 백업, 삭제, 업그레이드

K3S는 가볍고 설치/운영관리가 편리한 공인된 쿠버네티스배포판
- 빠른 설치 (Online, Airgapped)
- 쉬운 백업 및 복구
- 간편한 버전업그레이드 및 삭제
- 인그레스, 스토리지클래스, 로드밸런서 등 클러스터 서비스 기본 제공

> https://docs.k3s.io/kr/

### 0. 실습 환경

##### 0.1 서버 접속
```
- **** / root / 8022
- lab.key

$ ssh -i lab.key -p 8022 root@****

```

##### 0.2 서버 내 VM 4기 구성 libvirt 

```
- node-01 / 192.168.122.11 / root ssh config 설정됨 / ssh node-01 로 접속 
- node-02 / 192.168.122.12
- node-03 / 192.168.122.13
- node-04 / 192.168.122.14
```

### 1. K3S 클러스터 설치

##### 1.1 단일 마스터 노드

```
# 노드 1번 접속
$ ssh node-01

# K3S 마스터 노드 설치 (embedded etcd)
$ curl -sfL https://get.k3s.io | sh -s - --cluster-init

```

##### 1.2 2노드 - 마스터 / 워커

```
# 노드 2번 접속
$ ssh node-02 

# K3S 마스터 노드 토큰 조회
$ export master_ip=192.168.122.11
$ export token=$(ssh node-01 cat /var/lib/rancher/k3s/server/token)

# K3S 마스터 노드에 워커 노드 추가
$ curl -sfL https://get.k3s.io | K3S_URL=https://$master_ip:6443 K3S_TOKEN=$token sh -s -

```

##### 1.3 클러스터 확인

```
# 노드 1번 접속
$ ssh node-01

# kubectl 파드, 노드 확인
$ kubectl get nodes
$ kubectl get pods -A

# k3s 서비스 확인
$ systemctl status k3s.service

# kubeconfig 확인
$ cat /etc/rancher/k3s/k3s.yaml

```

##### 1.4 클러스터 스냅샷 생성

```
# 노드 1번 접속
$ ssh node-01

# etcd snapshot 생성
$ k3s etcd-snapshot save

# etcd snapshot 조회
$ k3s etcd-snapshot ls

```

##### 1.4 K3S 삭제

```
# 노드 1번 접속
$ ssh node-01

# k3s 마스터 노드 삭제
$ k3s-uninstall.sh

# 노드 2번 접속
$ ssh node-02

# k3s 워커 노드 삭제
$ k3s-agent-uninstall.sh

```

##### 1.5 K3S 업그레이드

```
# 노드 1번 접속
$ ssh node-01

# SELINUX 설정 변경
$ setenforce 0

# k3s 종료 및 백업
$ systemctl stop k3s.service
$ mv /usr/local/bin/k3s /usr/local/bin/k3s-1.30.5

# k3s 신규 버전 다운로드
$ curl -LO https://github.com/k3s-io/k3s/releases/download/v1.31.1%2Bk3s1/k3s
$ chmod +x k3s && mv k3s /usr/local/bin

# k3s 서비스 기동
$ systemctl start k3s.service

# kubernetes 버전 확인
$ kubectl get nodes

```
