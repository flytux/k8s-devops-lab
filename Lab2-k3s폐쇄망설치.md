### 1. K3S 폐쇄망 설치

##### 1.1 docker hub 접속 불가 설정
```
# 도커 허브 주소 설정
$ echo "0.0.0.0 index.docker.io auth.docker.io registry-1.docker.io" >> /etc/hosts

# 접속 테스트
$ docker pull nginx
```

##### 1.2 k3s 설치 파일 다운로드

k3s airgapped 설치를 위한 준비 파일은 다음과 같다.

1) install 스크립트
```
$ curl -sfL https://get.k3s.io > install.sh
$ chmod +x install.sh
```

3) k3s 바이너리 (https://github.com/k3s-io/k3s/releases)
```
$ curl -LO https://github.com/k3s-io/k3s/releases/download/v1.31.1%2Bk3s1/k3s

```

4) k3s 바이너리 버전 별 도커 컨테이너 이미지
```
$ curl -LO https://github.com/k3s-io/k3s/releases/download/v1.31.1%2Bk3s1/k3s-airgap-images-amd64.tar.gz
```

5) k3s 이미지 sha256 검증용 파일
```
$ curl -LO https://github.com/k3s-io/k3s/releases/download/v1.31.1%2Bk3s1/sha256sum-amd64.txt
```
   
selinux가 enforce 되어 있는 경우, 해당 OS 버전의 selinux 패키지 설치 필요

5) k3s selinux policy 파일
```
$ https://github.com/k3s-io/k3s-selinux/releases
$ curl -LO https://github.com/k3s-io/k3s-selinux/releases/download/v1.6.stable.1/k3s-selinux-1.6-1.el8.noarch.rpm
- 동일한 OS 버전의 selinux 의존 패키지를 먼저 다운로드 해 놓는다.
$ yum install ./k3s-selinux-1.6-1.el8.noarch.rpm --downloadonly --downloaddir=. 
```
##### 1.3 k3S 설치 - 첫째 마스터 노드

```
# 이미지 디렉토리 생성
$ chmod +x k3s && cp k3s /usr/local/bin
$ sudo mkdir -p /var/lib/rancher/k3s/agent/images/
$ cp k3s-airgap-images-amd64.tar.gz sha256sum-amd64.txt /var/lib/rancher/k3s/agent/images/
```

```
# default route 설정
$ ip route | grep default
   
(*) 결과가 없는 경우 dummy route 생성
$ ip link add dummy0 type dummy
$ ip link set dummy0 up
$ ip addr add 203.0.113.254/31 dev dummy0
$ ip route add default via 203.0.113.255 dev dummy0 metric 1000
```

```
# selinux rpm 설치
# RPM 패키지가 있는 폴더에서 설치한다.
$ rpm -Uvh *.rpm
```   

```
# k3s 마스터 시작 노드 설치 
# k3s config 파일을 이용하여 다양한 클러스터 설정을 적용한다.
$ cat << EOF >>  vi /etc/rancher/k3s/config.yaml
  cluster-init: true # 내장 etcd 사용 설정
EOF
$ INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh
```

##### 1.4 k3S 설치 - 워커노드 

```
# node-01에 다운받은 설치 파일을 node-02로 복사한다.
$ ssh node-01
$ tar cvzf k3s-airgapped-install.tgz install.sh container-selinux* k3s
$ scp k3s-airgapped-install.tgz node-02:/root

$ ssh node-02
$ tar xvf k3s-airgapped-install.tgz

# 설치 파일을 해당 디렉토리에 복사하고 실행 권한을 설정한다.
$ chmod +x k3s && cp k3s /usr/local/bin
$ sudo mkdir -p /var/lib/rancher/k3s/agent/images/
$ cp k3s-airgap-images-amd64.tar.gz sha256sum-amd64.txt /var/lib/rancher/k3s/agent/images/

$ rpm -Uvh *.rpm

# k3s worker 노드를 구동한다.
$ export master_ip=192.168.122.11
$ export token=$(ssh node-01 cat /var/lib/rancher/k3s/server/token)

# K3S 마스터 노드에 워커 노드 추가, K3S_URL을 환경변수로 설정하면 k3s-agent 실행 = 워커노드
$ INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://$master_ip:6443 K3S_TOKEN=$token ./install.sh
```

##### 1.5 k3S 설치 - 두번째 마스터 노드 

```
# 필요한 파일을 노드에 복사하고 k3s 설치 시, K3S 서버를 argument로 추가하면 k3s-server 실행 = 마스터 노드 추가
$ INSTALL_K3S_SKIP_DOWNLOAD=true K3S_TOKEN=$token ./install.sh --server=https://$master_ip:6443
```
