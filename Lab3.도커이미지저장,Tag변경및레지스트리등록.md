##### 1. 컨테이너 이미지 확인

```
# k3s가 설치된 상태에는 ctr, crictl 을 이용하여 이미지 조회가 가능
# nerdctl을 설치하여 도커와 동일한 명령어 실행

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
# TO-DO
# helm 설치
# 인증서 생성
# containerd에 registry 인증서 등록
```


##### 3. 이미지 목록 확인

```
nerctl images --format "{{.Repositorty}}:{{.Tag}}" > images.txt
```

##### 4. Tag가 없는 이미지는 확인하여 Tag 작성


##### 5. 비어있는 이미지 가져오기

```
cat images.txt | while read line
do
nerdctl pull ${line}
done
```

##### 6. 이미지 tar로 저장

```
nerdctl images | grep -v REPOSITORY | grep -v none | while read line
do
  filename=$( echo "$line" | awk '{print $1":"$2".tar"}' | sed 's|:|@|g ; s|/|+|g' )
  option=$( echo "$line" | awk '{print $1":"$2}' )
  echo "nerdctl save ${option} -o ${filename}"
  nerdctl save "${option}" -o "${filename}"
done
```

##### 7. 이미지 로드

```
ls *.tar | while read line
do
   filename=$( echo "$line" )
   echo "nerdctl load -i ${filename}"
   nerdctl load -i "${filename}"
done
```
