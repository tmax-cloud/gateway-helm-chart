# API-GATEWAY

Api-gateway is a HTTP reverse proxy and a ingress controller for tmax-cloud modules.

## Introduction

Api-gateway consist of a total of 4 charts.
- Cert-manager
- gateway (traefik) <-- Core Module
    - required: cert-manager
- jwt-decode-auth
    - required: hyperauth
- console
    - required: hyperauth

## Installing

### Prerequisites

With the command `helm version`, make sure that you have:
- Helm v3 [installed](https://helm.sh/docs/using_helm/#installing-helm)

### Deploying Api-Gateway
1. Deploying cert-manager
```shell
cd cert-manager
```
- Setting the values at values.yaml
```yaml
tags:
  certManager: true

cert-manager:
  installCRDs: true
  fullnameOverride: "cert-manager"
  extraArgs:
    - "--dns01-recursive-nameservers=8.8.8.8:53"
    - "--dns01-recursive-nameservers-only"

global:
  createCA: true
  sleepyTime: 120
# kubectl create secret docker-registry regcred --docker-username=tmaxcloudck --docker-password=$PASSWD 
# imagePullSecrets:
#  - name: regcred
```
```shell
helm install cert-manager . --namespace cert-manager --create-namespace      
```
2. Deploying gateway(traefik)
```shell
cd gateway
```
- Setting the values at values.yaml
```yaml
# Default values for gateway.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

tags:
  traefik: true

traefik:
  fullnameOverride: "gateway"
  additionalArguments:
    # Only use dev environment
    # - "--serverstransport.insecureskipverify=true"
    - "--serverstransport.rootcas=/run/secrets/tmaxcloud/ca.crt, /run/secrets/kubernetes.io/serviceaccount/ca.crt"
    - "--entrypoints.websecure.http.middlewares=cors-header@file"
  deployment:
    podLabels:
      app: traefik
    imagePullSecrets: []
  #    imagePullSecrets:
  #      - name: regcred
  volumes:
    - name: gateway-config
      mountPath: "/gateway-config"
      type: configMap
    - name: selfsigned-ca
      mountPath: "/run/secrets/tmaxcloud"
      type: secret
  ingressClass:
    enabled: true
    isDefaultClass: true
    fallbackApiVersion: "v1"
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-internal: "true"
  ## When using NodePort service
  #    type: NodePort
  #  ports:
  #    k8s:
  #      nodePort: 31643
  #    web:
  #      nodePort: 31080
  #    websecure:
  #      nodePort: 31433
  #    traefik:
  #      nodePort: 31900

  resources: {}
  nodeSelector:
    node-role.kubernetes.io/master: ""
  tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: Exists
tls:
  domain: tmaxcloud.test
  acme:
    enabled: false
    email: email
    dns:
      type: route53
      accessKeyID: accesskey
      accessKeySecret: secretkey
      hostedZoneID: Z07854445SLZAKEQ0V9P
    environment: staging
  selfsigned:
    enabled: true
```
```shell
helm install api-gateway . --namespace api-gateway-system --create-namespace 
```
3. Deploying jwt-decode-auth (required hyperauth address, realm)
```shell
cd jwt-decode-auth 
```
- Setting the values at values.yaml
```yaml
hyperauth:
  address: hyperauth.org
  realm: tmax
middleware:
  enabled: true
```
```shell
helm install jwt-decode-auth . --namespace api-gateway-system
```
4. Deploying console (required hyperauth address, realm)
```shell
cd console
```
- Setting the values at values.yaml
```yaml
nameOverride: "console"
fullnameOverride: "console"

console:
  auth:
    hyperauth: hyperauth.org
    realm: tmax
    clientid: hypercloud5
  mcMode: false

ingressroute:
  enabled: true
  domain:
    enabled: ture
    name: tmaxcloud.org
  ip:
    enabled: false
```
```shell
helm install console . --namespace api-gateway-system
```