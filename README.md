*인증서 가격이 상당히 비쌉니다.
![image.png](/.attachments/image-229e1649-39cb-4140-8b4d-08c02d70b4ac.png)

# aks Cluster 생성 
![image.png](/.attachments/image-444f7fdb-7b6f-498f-9a39-d9bc28e25dff.png)

## internal LB생성을 위해 subnet에 권한 추가
![image.png](/.attachments/image-f90d0e59-6992-4ac9-b796-767cab98ac8f.png)
![image.png](/.attachments/image-b246422a-221a-4b5b-a485-eed8b4ac9c9c.png)

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
![image.png](/.attachments/image-89cdc640-a3b8-4517-91d1-fa999fd71852.png)


# Application gateway 용 pip생성

## frontend pip DNS name label 설정

![image.png](/.attachments/image-be2d2f73-93ce-4f9c-916d-10e803f8bfad.png)

*DNS Label 중복되지 않는 값 설정 필요
**개인 DNS보유 시 a record 등록 후 사용 가능

# 인증서 발급
1, 2 선택
##1. LetsEncrypt
certbot 설치(https://certbot.eff.org/lets-encrypt/windows-other.html)

```
## power shell administroator권한 필요
## 인증서 발급 (challenge 필요) 추가 가이드
certbot certonly --manual --email <user email> -d <Your Domain>
```
아래 창에서 대기
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

<Validation Code>

And make it available on your web server at this URL:

http://<Your Domain>/.well-known/acme-challenge/<File Name>

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```


##2. app service certificate 발급
SKU : Standard
Naked domain hostname: <Your Domain>(DNS Label 주소 사용)

![image.png](/.attachments/image-97822ddb-2519-4e19-814f-6acd34164a8d.png)



# Application gateway 생성

## backend pool
rancher-backend : 10.0.1.250
cert-validation : 10.0.1.251

## rules

![image.png](/.attachments/image-8725d5cb-8eaa-4cda-bd27-b29b207dc34c.png)

path based rule 설정으로 다음과 같이 설정
- path : /.well-known/*
- target name : cert-validation
- http settings : http
- Backend target :cert-validation

![image.png](/.attachments/image-9c579840-fe27-4414-9d6a-ab1eec59be5b.png)


## frontend ip : dns label 생성한 Pip 연결

## backend target : rancher-backend
![image.png](/.attachments/image-2a8b2cd3-b517-414f-b0dd-92b25c2c8bef.png)

# certificate validation을 위해 aks에 앱 생성

인증서 발급 시 선택한 방법 사용

## 1. letsencrypt certificate
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: letsencript-challenge
  labels:
    app: challenge
data:
  <File Name>: |
    <Validation Code>

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
인증서 발급 확인(Letsecrypt 인증서 발급 시 대기화면에서 엔터 입력)
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

<Validation Code>

And make it available on your web server at this URL:

http://<Your Domain>/.well-known/acme-challenge/<File Name>

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   C:/Certbot/live/<Your Domain>/fullchain.pem    ---> 인증서 위치 표시
   Your key file has been saved at:
   C:/Certbot/live/<Your Domain>/privkey.pem
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
pfx 인증서 생성([open ssl](https://www.xolphin.com/support/OpenSSL/OpenSSL_-_Installation_under_Windows) 설치 필요)

```
cd C:/Certbot/live/<Your Domain>
openssl pkcs12 -export -out certificate.pfx -inkey privkey.pem -in cert.pem -certfile chain.pem
```
*Password 설정 필요
```
Enter Export Password:
Verifying - Enter Export Password:
```


pfx 파일 확인
![image.png](/.attachments/image-1fc133a9-e8d2-45d2-918c-bf7fc1246169.png)



## 2. Azure App service Certificate
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: challenge-index
  labels:
    app: challenge
data:
  godaddy.html: |
    <Validation Code>

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


## 인증 앱 배포 확인
![image.png](/.attachments/image-ac1d9204-e246-4cf1-98ed-5e434688955b.png)

# 인증서 verify 확인
![image.png](/.attachments/image-40b89378-c3d7-484d-acac-3d093e6c460c.png)

# azure key vault 생성해 인증서 store

![image.png](/.attachments/image-dccdd289-16c6-40eb-83b1-b7f460743df7.png)
# key vault에서 인증서 확인

![image.png](/.attachments/image-f483de05-9e7c-4486-a128-2036b54107fd.png)

## Secret Idetntifier 복사

![image.png](/.attachments/image-746740d8-9014-49a2-bf30-d6154d10900c.png)

# Application gateway에 인증서 등록을 위해 Managed ID 생성
![image.png](/.attachments/image-3a918366-370c-411e-9368-47681ed5fc65.png)

## key vault에서 add access policy로 managed id 등록 및 secret 권한에 get추가
![image.png](/.attachments/image-5ef01828-0d16-444d-b501-ceb3a5b877c7.png)

## 하단 user권한에 본인 계정 권한 중 secret, certificate 퍼미션 list권한 제거
![image.png](/.attachments/image-43486963-802c-40a8-a6cb-2b95756bddec.png)


# application gateway https-listener추가
## https-listener생성
## 1. letsencrypt
pfx파일 업로드
Password : 인증서 변환시 입력한 패스워드 입력
![image.png](/.attachments/image-5019143e-7991-4a6e-90e3-c35152c9c42f.png)

## 2. App service certificate
- managed id : 위에서 생성한 managed id
- key vault : 인증서가 들어있는 키볼트
- Certificate : 위에서 복사한 Secret Idetntifier

*만약 certficate가 콤보박스 형태로 나온다면 keyvault에서 [유저권한 list퍼미션](https://dev.azure.com/k8scloocusstudy/K8s%20Study/_wiki/wikis/K8s-Study.wiki/81/aks-cluster%EC%97%90-Rancher%EC%84%A4%EC%B9%98(azure-app-service-%EC%9D%B8%EC%A6%9D%EC%84%9C-ssl-Termination-%EC%82%AC%EC%9A%A9)?anchor=%ED%95%98%EB%8B%A8-user%EA%B6%8C%ED%95%9C%EC%97%90-%EB%B3%B8%EC%9D%B8-%EA%B3%84%EC%A0%95-%EA%B6%8C%ED%95%9C-%EC%A4%91-secret%2C-certificate-%ED%8D%BC%EB%AF%B8%EC%85%98-list%EA%B6%8C%ED%95%9C-%EC%A0%9C%EA%B1%B0) 제거 필요

![image.png](/.attachments/image-3b04f763-b6ee-443f-a695-8c2181c317e0.png)

# https 규칙 생성
![image.png](/.attachments/image-4a2fc0b0-c78d-4ee1-b8ab-ffb619d6250f.png)
![image.png](/.attachments/image-fe07b157-a868-408a-8418-add2d5acb57c.png)

# http -> https 리다이렉션 설정
## 기존 http rule에서 백엔드 타겟 변경
- target type : redirection
- redirection type : Permanent
- redirrection target : Listener
- target Listner https-listener

![image.png](/.attachments/image-279b1079-64e7-413b-8103-9be23f083df4.png)

# Rancher 접속 확인
## DNS Label로 세팅한 url로 접속
![image.png](/.attachments/image-cfa463b1-3a1b-4d7d-8443-316e072ba94a.png)

# worker클러스터 생성
![image.png](/.attachments/image-c8e84714-103f-4f1a-99ab-2d7fdf773bfe.png)

# rancher에 import
![image.png](/.attachments/image-3bf61438-d04a-4fc7-8b5c-5ad546055205.png)

![image.png](/.attachments/image-4fae6e5b-bfb9-4dbe-b5d1-73274e31c60e.png)

![image.png](/.attachments/image-ac325d36-2ecd-4a5b-b0f1-a0162abc1354.png)

![image.png](/.attachments/image-7484db7a-9c1c-4779-83ac-74444220e7c8.png)

## worker cluster에 설치
![image.png](/.attachments/image-23f4362b-0f87-4213-a37f-6372b6d5a39c.png)

## worker cluster imported
![image.png](/.attachments/image-e46e9baa-c70a-4a14-946a-b3dad3d192f0.png)


https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/
https://www.godaddy.com/help/verify-domain-ownership-html-or-dns-for-my-ssl-certificate-7452
https://docs.microsoft.com/ko-kr/azure/app-service/configure-ssl-certificate#import-an-app-service-certificate
https://lemontia.tistory.com/865
