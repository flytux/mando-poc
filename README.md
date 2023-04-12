# mando-poc

## Rancher 실습


*) 사전 준비 사항

1) linux VM 
2) 외부망 인터넷 접속 가능 
3) rancher 서버 (43.200.71.229) 80/443 접속 가능 확인 
4) rancher 서버 hosts 추가

cat <<EOF >> /etc/hosts
43.200.71.229 rancher.m2mcx01-public
EOF

5) (Optional) NetworkManager 설치 구동 중인 경우 cni 서비스 예외처리 

sudo -i
mkdir -p /etc/NetworkManager/conf.d
cat <<EOF >> /etc/NetworkManager/conf.d/rke2-canal.conf
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
EOF

### 1. RKE2 설치

1) rke2 설치 

curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server --now &
systemctl status -l rke2-server

2) kubectl 설치 및 설정

curl -LO https://dl.k8s.io/release/v1.24.10/bin/linux/amd64/kubectl
chmod 755 kubectl && mv kubectl /usr/local/bin
mkdir -p ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
kubectl get nodes
   
cat <<EOF >> ~/.bashrc
   source <(kubectl completion bash)
   complete -o default -F __start_kubectl k
   alias k=kubectl
   alias kn='kubectl config set-context --current --namespace'
   alias kc='kubectl config use-context'
   alias kcg='kubectl config get-contexts'
   alias kpa='kubectl get pods -A'
EOF

### 2. 랜처 연결

https://rancher.m2mcx01-public
M2MCx / Mando1234!@#$

- 메인 화면에서 Import Existing 선택
- Import any Kubernetes cluster > Generic 선택
- Cluster Name : 자기 클러스터 명 입력 > Create
- 2번째 설치 명령어 복사 : "curl --insecure ... "
- VM에서 해당 복사 명령어 실행

ex) curl --insecure -sfL https://rancher.m2mcx01-public/v3/import/vtd64wcrxzwvx67k552grbhdw6tpwshpj252h2tt94krzmnpgmvqcx_c-m-r5b9rn29.yaml | kubectl apply -f -

- 디플로이먼트 확인

kn cattle-system
k get pods

- host Alias 패치
cat <<EOF >> agent-patch.yml
spec:
  template:
    spec:
      hostAliases:
      - ip: 43.200.71.229
        hostnames:
        - rancher.m2mcx01-public
EOF

kubectl patch deployment cattle-cluster-agent --patch-file agent-patch.yml


- 랜처 연결 확인


### 3. 사용자 계정 설정

- 랜처 로그인
- 메뉴 > Configuration > Users & Authentication
- Create > Username > 사용자 이름 입력 > Password 입력
- Global Permissions > Standard User > Create > 로그아웃 후 새 사용자로 로그인
- M2MCx 로그인 > 메뉴 > Cluster Management > 추가된 클러스터 ... 메뉴 > Edit Config
- Member Roles > Add Member > Name - 새 사용자 입력 > Role - Owner 선택 > Save > 로그아웃
- 새 사용자 로그인 > 클러스터 내용 확인

### 4. ArgoCD App 등록

- 신규 클러스터 kubectl context 설정 : AWS rancher VM (Rancher Generated Kubeconfig 이용)
- ArgoCD 로그인
- ArgoCD add cluster
- ArgoCD app 추가
- 이미지 빌드/푸시 시 ArgoCD Deploy 자동 호출
- deploy-kust-dev 네임스페이스 배포 확인
- ImagePullSecret 확인 없으면 생성

### 5. Logging Operator 설정

- Logging Operator 설치
- Log Output 생성 (loki, s3 output)
- Log Flow 생성
- Loki / Minio 로그 확인
- S3 로그를 Grafana로 모니터링 하기 위해서는 
  1) hive 등을 통해 Time Series로 변환 후 Loki / infuxdb 등에 적재하여 연결하거나
  2) S3 > Lamda Function es-loader > AWS OpenSearch 에 적재 필요 (https://catalog.us-east-1.prod.workshops.aws/workshops/60a6ee4e-e32d-42f5-bd9b-4a2f7c135a72/en-US/05-ingest-and-process-application-logs)

### 6. Neuvector 보안 점검

- Neuvector 설치
- Neuvector 노드 / 컨테이너 점검
