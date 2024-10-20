### Lab4. RKE2 폐쇄망 설치, 백업, 삭제, 업그레이드

- RKE2는 기본 엔진을 k3s와 동일하게 사용
- 기본적으로 k3s 폐쇄망 설치와 동일한 방식으로 수행
- rke2 버전별 바이너리와 도커 이미지 파일, 인스톨 스크립트를 받아서 인스톨 스크립트를 실행하여 수행 (https://docs.rke2.io/install/airgap)


##### 1) 설치 파일 다운로드
```
$ mkdir /root/rke2-artifacts && cd /root/rke2-artifacts/
$ curl -OLs https://github.com/rancher/rke2/releases/download/v1.30.5%2Brke2r1/rke2-images.linux-amd64.tar.zst
$ curl -OLs https://github.com/rancher/rke2/releases/download/v1.30.5%2Brke2r1/rke2.linux-amd64.tar.gz
$ curl -OLs https://github.com/rancher/rke2/releases/download/v1.30.5%2Brke2r1/sha256sum-amd64.txt
$ curl -sfL https://get.rke2.io --output install.sh

```

##### 2) 마스터 노드 설치
```
$ mkdir -p /etc/rancher/rke2
$ cat << EOF > /etc/rancher/rke2/config.yaml
token: rke2installtoken # 클러스터 서버 토큰
tls-san:
  - rke2-master
EOF

$ INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh install.sh
$ systemctl enable rke2-server.service --now
```

##### 3) 마스터 노드 2nd 설치
```
# 설치 파일 2nd 노드에 복사
$ tar cvzf rke2-install.tgz rke* sha* install.sh
$ scp rke2-install.tgz node-02:/root

$ mkdir -p /etc/rancher/rke2
$ cat << EOF > /etc/rancher/rke2/config.yaml
server: https://192.168.122.11:9345
token: rke2installtoken # 클러스터 서버 토큰
tls-san:
  - rke2-master
EOF

$ systemctl enable rke2-server.service --now
```

##### 4) 워커 노드 설치
```
# 설치 파일 워커노드에 복사
$ tar cvzf rke2-install.tgz rke* sha* install.sh
$ scp rke2-install.tgz node-03:/root

$ mkdir -p /etc/rancher/rke2
$ cat << EOF > /etc/rancher/rke2/config.yaml
server: https://192.168.122.11:9345
token: rke2installtoken # 클러스터 서버 토큰
tls-san:
  - rke2-master
EOF

$ INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh install.sh
$ systemctl enable rke2-agent.service --now
```

```
# node-01에 접속하여 노드 정보 확인
$ ssh node-01
$ kubectl get nodes
```

##### 5) 노드 삭제
```
$ rke2-uninstall.sh
```

##### 6) 노드 업그레이드
```
# rke2 신규 버전을 다운로드하여 /usr/local/bin/rke2로 복사 (이전 버전 백업)
$ curl -LO https://github.com/rancher/rke2/releases/download/v1.31.1%2Brke2r1/rke2.linux-amd64

# 해당 버전의 컨테이너 이미지를 로컬 저장소에 다운로드 후 복사
$ curl -LO https://github.com/rancher/rke2/releases/download/v1.31.1%2Brke2r1/rke2-images.linux-amd64.tar.gz
$ cp rke2-images.linux-amd64.tar.gz /var/lib/rancher/rke2/agent/images/

$ systemctl stop rke2-server
$ mv /usr/local/bin/rke2 /usr/local/bin/rke2-1.30.5
$ chmod +x rke2.linux-amd64 && mv rke2.linux-amd64 /usr/local/bin/rke2

$ systemctl start rke2-server
```

##### 7) 노드 삭제
```
$ rke2-uninstall.sh
```

##### 8) 노드 백업
```
$ rke2 etcd-snapshot save
$ rke2 etcd-snapshot ls
```


