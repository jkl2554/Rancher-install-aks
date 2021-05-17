*인증서 가격이 상당히 비쌉니다.
# aks Cluster 생성 
- Loadbalancer : standard
- Authentication method : System-assigned managed identity
- node pool subnet : 10.0.1.250, 10.0.1.251 이 포함된 서브넷(예: 10.0.1.0/24)

## internal LB생성을 위해 subnet에 권한 추가
- role: Network Contributor
- select: \<aks Cluster name\>
- role: reader
- select: \<aks Cluster name\>

# rancher 설치 (helm 차트)
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

kubectl create ns cattle-system

helm install rancher rancher-stable/rancher `
  --namespace cattle-system `
  --set ingress.enabled="false" `
  --set tls=external
```

# rancher 접속 용 service 생성
```
apiVersion: v1
kind: Service
metadata:
  namespace: cattle-system
  name: rancher-internal
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: rancher
  sessionAffinity: None
  type: LoadBalancer
  loadBalancerIP: 10.0.1.250

```
# Application gateway 용 pip생성
- frontend pip DNS name label 설정

*DNS Label 중복되지 않는 값 설정 필요  
**개인 DNS보유 시 a record 등록 후 사용 가능  

# Application gateway 생성
## backend pool
- rancher-backend : 10.0.1.250
- cert-validation : 10.0.1.251

## rules
path based rule 설정으로 다음과 같이 설정
- path : /.well-known/*
- target name : cert-validation
- http settings : http
- Backend target :cert-validation

## frontend ip 
- dns label 생성한 Pip 연결

## backend target 
- rancher-backend

# 인증서 발급
*1, 2 선택

## 1. LetsEncrypt
certbot 설치(https://certbot.eff.org/lets-encrypt/windows-other.html)

```
## power shell administroator권한 필요
## 인증서 발급 (challenge 필요) 추가 가이드
certbot certonly --manual --email \<user email\> -d \<Your Domain\>
```
아래 창에서 대기
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

\<Validation Code\>

And make it available on your web server at this URL:

http://\<Your Domain\>/.well-known/acme-challenge/\<File Name\>

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```
### certificate validation을 위해 aks에 앱 생성
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: letsencript-challenge
  labels:
    app: challenge
data:
  \<File Name\>: |
    \<Validation Code\>

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redirectpage
spec:
  selector: 
    matchLabels:
      app: challenge
  replicas: 1
  template:
    metadata:
      labels:
        app: challenge
    spec:
      containers:
      - name: name
        image: httpd
        ports:
        - containerPort: 80
        volumeMounts:
          - name: letsencript-challenge
            mountPath: "/usr/local/apache2/htdocs/.well-known/acme-challenge/"
      volumes:
      - name: letsencript-challenge
        configMap:
          name: letsencript-challenge
---
kind: Service
apiVersion: v1
metadata:
  name: challengefront
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  loadBalancerIP: 10.0.1.251
  selector:
    app:  challenge
  type:  LoadBalancer
  ports:
  - name:  challengeport
    port:  80
    targetPort:  80
```
### 인증 앱 배포 확인
```
curl http://\<Your Domain\>/.well-known/acme-challenge/\<File Name\>
\<Validation Code\>
```

인증서 발급 확인(Letsecrypt 인증서 발급 시 대기화면에서 엔터 입력)
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

\<Validation Code\>

And make it available on your web server at this URL:

http://\<Your Domain\>/.well-known/acme-challenge/\<File Name\>

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   C:/Certbot/live/\<Your Domain\>/fullchain.pem    ---\> 인증서 위치 표시
   Your key file has been saved at:
   C:/Certbot/live/\<Your Domain\>/privkey.pem
   Your cert will expire on 2019-07-20. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
### pfx 인증서 생성([open ssl](https://www.xolphin.com/support/OpenSSL/OpenSSL_-_Installation_under_Windows) 설치 필요)

```
cd C:/Certbot/live/\<Your Domain\>
openssl pkcs12 -export -out certificate.pfx -inkey privkey.pem -in cert.pem -certfile chain.pem
```
*Password 설정 필요
```
Enter Export Password:
Verifying - Enter Export Password:
```
pfx 파일 확인
```
ls certificate.pfx
```
### azure key vault 생성해 인증서 upload
- certificate - + Genterate/Import
- Method of Certificate Creation: Import
- Certificate Name: \<cert name\>
- Upload Certificate File: C:/Certbot/live/\<Your Domain\>/certificate.pfx
- Password: pfx인증서 생성시 입력한 패스워드 입력

### Application gateway에 인증서 등록을 위해 Managed ID 생성
- Managed Identity 생성
- key vault - add access policy: managed id 등록 및 certificate 권한에 get추가

### application gateway https-listener추가
- Choose a certificate: Choose a certificate from Key Vault
- Managed identity: 위에서 생성한 Managed Identity
- Key Vault: 생성한 Key vault
- Certificate \<cert name\>

*pfx파일을 직접 업로드 해 Application gateway 인증서 생성 가능


## 2. app service certificate 발급
- SKU : Standard
- Naked domain hostname: \<Your Domain\>(DNS Label 주소 사용)  
*Wild Card 도메인 사용 시 변경 필요.

### 인증서 verify 확인
- Certificate Configuration - Verify
- \<Validation Code\> 

### certificate validation을 위해 aks에 앱 생성
*자신의 도메인이라면 txt레코드를 통해 인증 가능
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: challenge-index
  labels:
    app: challenge
data:
  godaddy.html: |
    \<Validation Code\>

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redirectpage
spec:
  selector: 
    matchLabels:
      app: challenge
  replicas: 1
  template:
    metadata:
      labels:
        app: challenge
    spec:
      containers:
      - name: name
        image: httpd
        ports:
        - containerPort: 80
        volumeMounts:
          - name: challenge-index
            mountPath: "/usr/local/apache2/htdocs/.well-known/pki-validation/"
            readOnly: true
      volumes:
      - name: challenge-index
        configMap:
          name: challenge-index
---
kind: Service
apiVersion: v1
metadata:
  name: challengefront
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  loadBalancerIP: 10.0.1.251
  selector:
    app:  challenge
  type:  LoadBalancer
  ports:
  - name:  challengeport
    port:  80
    targetPort:  80
```
### 인증 앱 배포 확인
```
curl http://\<Your Domain\>/.well-known/pki-validation/godaddy.html
\<Validation Code\>
```

### azure key vault 에 인증서 store
- Azure Key Vault 생성
- Application gateway에 인증서 등록을 위해 Managed ID 생성
- Certificate Configuration - Store
- 생성한 Key vault 선택
- Key Vault - Secrets에서 인증서 확인(\<cert Name\>\<uuid\>, type application/x-pkcs12)
- 인증서에 들어가서 Secrets Idetntifier 복사
- key vault에서 add access policy로 managed id 등록 및 secret 권한에 get추가
- 하단 user권한에 본인 계정 권한 중 secret, certificate 퍼미션 list권한 제거


## application gateway https-listener추가
- managed id : 위에서 생성한 managed id
- key vault : 인증서가 들어있는 Key Vault
- Certificate : 위에서 복사한 Secret Idetntifier

*만약 certficate가 콤보박스 형태로 나온다면 keyvault에서 유저권한 list퍼미션 제거 필요

## https 규칙 생성
- Listener : https-listener
- target type : Backend pool
- Backend target : rancher-backend
- http settings : http
## 기존 http rule에서 백엔드 타겟 변경(http -\> https 리다이렉션 설정)
- Listener : http-listener
- target type : redirection
- redirection type : Permanent
- redirection target : Listener
- target Listner https-listener

# Rancher 접속 확인
## DNS Label로 세팅한 url로 접속
- admin 패스워드 설정 및 로그인
# rancher에 Cluster Import
## Worker AKS 클러스터 생성
## cluster Import
- Add Cluster - Registor an existing Kubernetes cluster - Other Cluster
- \<Cluster Name\> - Create
- Import Cluster창에서 보이는 코드 복사
## worker cluster에 설치
- worker cluster 생성 및 접속(kubeconfig)
- 위에서 복사한 코드로 배포
## worker cluster imported 확인


https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/
https://www.godaddy.com/help/verify-domain-ownership-html-or-dns-for-my-ssl-certificate-7452
https://docs.microsoft.com/ko-kr/azure/app-service/configure-ssl-certificate#import-an-app-service-certificate
https://lemontia.tistory.com/865
