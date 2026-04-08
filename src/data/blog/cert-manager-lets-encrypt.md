---
title: "Automating Certificate Management for Kubernetes using Cert-Manager and Let's Encrypt"
author: Murali Kotharamban
pubDatetime: 2021-05-20T00:00:00.000Z
slug: cert-manager-lets-encrypt
featured: false
draft: false
tags:
  - Kubernetes
  - TLS
  - cert-manager
  - Security
  - DevOps
description: "How to automate TLS certificate lifecycle management for Kubernetes applications using cert-manager and Let's Encrypt free certificates."
---

TLS certificate is one of the fundamental parts of modern web application security. Managing the lifecycle of TLS certificates for applications running in Kubernetes can be challenging — issuance, creating Kubernetes secrets, configuring ingress, tracking expiry and renewing on time, for every certificate in your environment. It's tedious and error-prone.

In this post, we'll see how to automate certificate management using Let's Encrypt's free certificates and cert-manager for applications running on Kubernetes.

We'll use the [Sock Shop](https://github.com/microservices-demo/microservices-demo) demo microservices application for this demonstration.

## Step 1: Install cert-manager

Cert-manager is an open source Kubernetes add-on that automates TLS certificate management. It manages the lifecycle of certificates issued by CAs — ensuring they're valid and renewed before expiry. It supports Let's Encrypt via the ACME protocol.

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

## Step 2: Install Contour Ingress Controller

We'll use Contour (a CNCF incubating project) as our ingress controller:

```bash
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

Verify the installation:

```bash
kubectl get all -n projectcontour
```

A successful install shows the contour deployment, envoy daemonset, and services. Note the external IP/hostname of the envoy LoadBalancer service — create a DNS record pointing your domain to it (CNAME for EKS, A record for GKE).

## Step 3: Deploy the Demo Application

```bash
git clone https://github.com/microservices-demo/microservices-demo.git
cd microservices-demo/deploy/kubernetes
kubectl create namespace sock-shop
kubectl apply -f complete-demo.yaml
```

## Step 4: Create a ClusterIssuer for Let's Encrypt

We'll create two issuers — staging for testing, production for real certificates. Let's Encrypt recommends staging first to avoid rate limits.

```yaml
# letsencrypt-staging-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: contour
```

```yaml
# letsencrypt-prod-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: contour
```

```bash
kubectl apply -f letsencrypt-staging-issuer.yaml
kubectl apply -f letsencrypt-prod-issuer.yaml
```

## Step 5: Create an Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: front-end-ingress
  namespace: sock-shop
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: contour
spec:
  tls:
  - hosts:
    - example.com
    secretName: sock-shop-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: front-end
            port:
              number: 80
```

Once applied, cert-manager detects the annotation, requests a certificate from Let's Encrypt, handles the HTTP01 challenge, and stores the cert in the `sock-shop-tls` secret automatically.

## Verify

```bash
kubectl get certificate -n sock-shop
kubectl describe certificate sock-shop-tls -n sock-shop
```

When the certificate status shows `Ready: True`, your application is serving HTTPS with an auto-renewing certificate. No manual intervention needed.

---

*Originally published on [Medium](https://muralidkt.medium.com/automating-certificate-management-using-cert-manager-and-lets-encrypt-51d883422fc1) — May 20, 2021*
