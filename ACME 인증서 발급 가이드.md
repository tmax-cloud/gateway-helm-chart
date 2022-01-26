## Cert-Manager를 통한 TLS 인증서 발급

### 1. AWS route53을 통한 유효한 인증서 발급
- 본 가이드는 cert-manager의 ACME 방식 중 DNS01을 이용하여 공인된 TLS 인증서 발급하는 방법에 대해 다룬다.
  - 참고 링크: [DNS01을 통한 인증서 개요](https://cert-manager.io/docs/configuration/acme/dns01/)
- DNS01방식에서 많은 DNS(Domain Name Service)를 통한 인증서 발급을 제공한다. 본 가이드는 그 중 AWS의 Route53을 통한 TLS 인증서 발급에 대해 다룬다. 
  - 참고 링크: [Route53을 통한 CLusterIssuer 생성](https://cert-manager.io/docs/configuration/acme/dns01/route53/), 
    [Route53에 대한 기본 설명](https://brunch.co.kr/@topasvga/49) 

#### STEP 0. 외부 CA를 통해 TLS 인증서를 발급받는 방식 & Client-Server간 https 통신 방식에 대한 기본 설명 
- 간략한 발급 방식 설명 
  - A 회사의 도메인(tmaxcloud.org)과 CSR(certificate signing request)를 CA 기관에 제출 
  - CA 기관에서 검토 (이 과정이 route53을 통한 인증 과정) 
  - A 회사의 CSR를 SHA-256등으로 해시한 후 인증서(tls.crt)에 지문으로 등록 
  - 등록한 지문을 CA의 개인키로 암호화 하여 인증서에 서명으로 등록 
  - 인증서 발급
- [참고 링크](https://babbab2.tistory.com/5)

#### STEP 1. AWS Console에서 IAM Role 설정과 Issuer 생성
목적: Cert-Manager에서 사용할 수 있도록 AWS Console에서 route53에 대한 권한 등록을 위한 과정이다.
- [해당 링크](https://voyagermesh.com/docs/v12.0.0/guides/cert-manager/dns01_challenge/aws-route53/) 에서 1.Setup Issuer를 참고하여 IAM role 등록 및 Issuer를 생성한다.

- ClusterIssuer 생성과 letsencrypt를 통한 공인된 TLS 인증서 발급을 위해 아래 yaml 파일을 참고하여 수정하길 권장한다. 
```yaml
apiVersion: certmanager.k8s.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    # tmaxcloud에서 사용하는 aws 계정 
    email: cloud_csp@tmax.co.kr
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
      - dns01:
          route53:
            accessKeyID: KIR2WO5YWT
            secretAccessKeySecretRef:
              name: route53-secret
              key: secret-access-key
            hostedZoneID: J13B3AB
```

#### STEP 2. Certificate을 사용하여 TLS secret 생성  
목적: Cert Manager에서 제공하는 Certificate 객체를 만들어서 우리가 사용할 TLS 인증서를 발급한다. 
- certificate.yaml 파일 생성 
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-acme-crt
  namespace: default # TLS 인증서가 secret 형태로 생성되길 원하는 네임스페이스  
spec:
  secretName: tls-acme-secret # secret 이름 설정 
  dnsNames:
    - tmaxcloud.org # route53에서 관리하고 있는 도메인 
    - "*.tmaxcloud.org" 
  issuerRef:
    name: letsencrypt-dns # 위에 생성한 clusterIusser와 동일 
    kind: ClusterIssuer
```

#### STEP 3. ingress-controller에 해당 인증서를 default 인증서로 사용 
- api-gateway(traefik) 일 경우: tlsstore.yaml을 생성한다. (traefik 설치되어 있다고 가정)
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: default # TLS 인증서가 secret 형태로 생성되길 원하는 네임스페이스  
spec:
  defaultCertificate:
    secretName: tls-acme-secret
```
- nginx-ingress-controller 일 경우: nginx-ingress-controller 파드의 arg를 추가한다. 
```yaml
containers:
- args:
    - --default-ssl-certificate=default/tls-acme-secret
```