---
title: "Automating Certificate Management using GKE's Managed Certificates"
author: Murali Kotharamban
pubDatetime: 2021-06-06T00:00:00.000Z
slug: gke-managed-certificates
featured: false
draft: false
tags:
  - GKE
  - Kubernetes
  - TLS
  - Google Cloud
  - Security
description: "How to use GKE Managed Certificates to automate TLS certificate lifecycle for applications running on Google Kubernetes Engine."
---

In a [previous post](/posts/cert-manager-lets-encrypt), I explored using Let's Encrypt and cert-manager to automate TLS certificate management on Kubernetes. In this post, we'll cover how to use **GKE Managed Certificates** to automate the TLS certificate lifecycle for applications running on Google Kubernetes Engine — no cert-manager required.

We'll use the [Sock Shop](https://github.com/microservices-demo/microservices-demo) demo application again.

> **Note:** At the time of writing, GKE Ingress does not support HTTP to HTTPS redirection when managed certificates are enabled.

## Step 1: Deploy the Application

```bash
git clone https://github.com/microservices-demo/microservices-demo.git
cd microservices-demo/deploy/kubernetes
kubectl create namespace sock-shop
kubectl apply -f complete-demo.yaml
```

Verify everything is running:

```bash
kubectl get pod -n sock-shop
kubectl get service -n sock-shop
```

## Step 2: Create a Managed Certificate

Update your domain name and apply the ManagedCertificate resource:

```yaml
# managed-cert.yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: sock-shop-certificate
  namespace: sock-shop
spec:
  domains:
    - example.com
```

```bash
kubectl apply -f managed-cert.yaml
```

The certificate status will show `Provisioning` until you attach it to an Ingress — that's expected. You can proceed to the next step.

```bash
kubectl get managedcertificates -n sock-shop
# NAME                    AGE   STATUS
# sock-shop-certificate   12m   Provisioning
```

## Step 3: Create an Ingress with the Managed Certificate

```yaml
# sock-shop-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: front-end-ingress
  namespace: sock-shop
  annotations:
    networking.gke.io/managed-certificates: sock-shop-certificate
    kubernetes.io/ingress.class: "gce"
spec:
  defaultBackend:
    service:
      name: front-end
      port:
        number: 80
```

```bash
kubectl apply -f sock-shop-ingress.yaml
```

## Step 4: Update DNS

Get the Ingress IP:

```bash
kubectl get ingress -n sock-shop
# NAME                CLASS    HOSTS   ADDRESS    PORTS   AGE
# front-end-ingress   <none>   *       X.X.X.X    80      20m
```

Create an **A record** in your DNS management pointing your domain to this IP.

## Step 5: Verify

Once DNS propagates and the certificate finishes provisioning, your application will be live on HTTPS. GKE handles the certificate issuance, renewal, and rotation automatically.

```bash
kubectl get managedcertificates -n sock-shop
# NAME                    AGE   STATUS
# sock-shop-certificate   30m   Active
```

Access your domain in a browser and verify the certificate — it should show a valid certificate issued by Google Trust Services.

---

*Originally published on [Medium](https://muralidkt.medium.com/automating-certificate-management-using-gkes-managed-certificates-31c30b2eaf21) — Jun 6, 2021*
