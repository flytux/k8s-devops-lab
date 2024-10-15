##### 1. 컨테이너 이미지 확인

```
# k3s가 설치된 상태에는 ctr, crictl 을 이용하여 이미지 조회가 가능
# nerdctl을 설치하여 도커와 동일한 명령어 실행

$ mkdir -p /etc/nerdctl
$ cat << EOF > /etc/nerdctl/nerdctl.toml
address        = "unix:///run/k3s/containerd/containerd.sock"
namespace      = "k8s.io"
EOF

$ curl -LO https://github.com/containerd/nerdctl/releases/download/v1.7.7/nerdctl-1.7.7-linux-amd64.tar.gz
$ tar Cxzvvf /usr/local/bin nerdctl-1.7.7-linux-amd64.tar.gz

$ ctr i ls
$ crictl images
$ nerdctl images
```

##### 2. 도커레지스트리 설치

```
# helm 설치
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# k9s 설치 및 kubeconfig 설정
$ wget https://github.com/derailed/k9s/releases/download/v0.27.4/k9s_Linux_amd64.tar.gz
$ tar xvf k9s_Linux_amd64.tar.gz && chmod +x k9s && sudo mv k9s /usr/local/bin
$ cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# 도커레지스트리용 인증서 생성
$ openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout docker.key -out docker.crt -subj '/CN=docker.local' \
  -addext 'subjectAltName=DNS:docker.local'

# 인증서 시크릿 생성
$ kubectl create ns registry
$ kubectl create secret tls docker-tls --key docker.key --cert docker.crt -n registry

# docker registry 설치
$ helm repo add twuni https://helm.twun.io

# ingress 설정
$ cat << EOF > values.yaml
ingress:
  enabled: true
  className: traefik
  hosts:
    - docker.local   
  tls:
    - secretName: docker-tls
      hosts:
        - docker.local
EOF

$ helm upgrade -i docker-registry -f values.yaml twuni/docker-registry -n registry

# 인증서 CA root 복사 Ubuntu
$ cp docker.key docker.crt /usr/local/share/ca-certificates/
$ update-ca-certificates

# 인증서 CA root 복사 RHEL
$ cp docker.key docker.crt /etc/pki/ca-trust/source/anchors/
$ update-ca-trust

$ cat << EOF >> /etc/hosts
192.168.122.11 docker.local # Ingress IP
EOF

cat << EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.local"
configs:
  "docker.local":
    auth:
      username: admin # this is the registry username
      password: q # this is the registry password
    tls:
      cert_file: /usr/local/share/ca-certificates/docker.crt # Ubuntu
      key_file: /usr/local/share/ca-certificates/docker.key  # Ubuntu
EOF

$ systemctl restart k3s-server # k3s-agent (워커노드)

# Docker registry 로그인
$ nerdctl login docker.local # admin / 1

```

#### 폐쇄망 내 클러스터 설치 시
- **필요한 도커 이미지를 외부에서 설치하여 확인하고**, 
- **Docker Save 명령을 통해 저장하여 반입한 후**,
- **Docker Load, Tag, Push 명령어를 이용하여 내부망 도커 레지스트리에 등록한다**.

##### 3. 이미지 목록 확인

```
nerdctl images --format "{{.Repository}}:{{.Tag}}" | grep -v none > images.txt
```

##### 4. Tag가 없는 이미지는 확인하여 Tag 작성

```
# image tag가 none인 경우
registry  <none>   dbaa3e69f563    11 minutes ago    linux/amd64    25.4 MiB     9.3 MiB

$ nerdctl tag dbaa3e69f563 registry:latest
```

##### 5. 이미지 목록에서 도커 이미지 가져오기

```
$ cat images.txt | while read line
do
  nerdctl pull ${line}
done
```

##### 6. 이미지 tar로 저장

```
$ nerdctl images | grep -v REPOSITORY | grep -v none | while read line
do
  filename=$( echo "$line" | awk '{print $1":"$2".tar"}' | sed 's|:|@|g ; s|/|+|g' )
  option=$( echo "$line" | awk '{print $1":"$2}' )
  echo "nerdctl save ${option} -o ${filename}"
  nerdctl save "${option}" -o "${filename}"
done

# 다음 에러 발생 시
# FATA[0000] failed to get reader: content digest sha256:2b1781f831: not found
# 오류난 이미지를
# nerdctl pull 컨테이너이미지:태그 --all-platforms 를 실행하여 전체 이미지 레이어를 가져온 후 저장을 다시 실행
```

##### 7. 이미지 로드

```
$ ls *.tar | while read line
do
  filename=$( echo "$line" )
  echo "nerdctl load -i ${filename}"
  nerdctl load -i "${filename}"
done
```

##### 8. 이미지 Tag 작성 및 이미지 Push

```
$ cat images.txt | while read line
do
  nerdctl tag ${line} docker.local/${line}
  nerdctl push docker.local/${line}
done

# 이미지 조회
$ curl -v docker.local/v2/_catalog
```
