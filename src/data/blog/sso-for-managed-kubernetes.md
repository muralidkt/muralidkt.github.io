---
title: "SSO for Managed Kubernetes Services (Part 1)"
author: Murali Kotharamban
pubDatetime: 2021-05-14T00:00:00.000Z
slug: sso-for-managed-kubernetes
featured: false
draft: false
tags:
  - Kubernetes
  - SSO
  - Okta
  - EKS
  - GKE
  - Security
description: "Setting up unified OIDC-based authentication across EKS and GKE using Okta as a single source of truth, eliminating duplicate identity management across cloud providers."
---

Recently I was working on setting up a unified authentication method for multiple managed Kubernetes clusters (EKS and GKE). My goal was to setup a seamless authentication experience for developers so that they don't need to worry about whether they are logging into clusters in GCP or AWS. Also I wanted to leverage my existing identity management solution (Okta) as a single source of truth for managing identities and avoid managing identities, groups, roles etc in cloud service provider's IAM.

As you know, cloud providers use their own IAM for authenticating users against managed Kubernetes offerings. AWS uses IAM principals like users and roles for EKS authentication. GKE manages authentication through Google Cloud users and service accounts. But in both cases we have to manage roles or groups in their native IAM services — which is redundant when you're already managing users and groups in Okta, and especially painful across multiple AWS accounts and GCP projects.

So I decided to explore OIDC authentication for Kubernetes to leverage Okta for authentication, Okta group membership, and Kubernetes RBAC for authorization across all clusters.

## Why OIDC for Managed Kubernetes is Tricky

Kubernetes natively supports OIDC authentication, but requires updating API Server configuration with parameters like:

```
--oidc-issuer-url
--oidc-client-id
--oidc-username-claim
--oidc-groups-claim
```

The whole point of managed Kubernetes is that the cloud provider manages the control plane — so they won't let you directly update the API Server config. However, AWS now supports integration with external OIDC providers, making it easier to associate an Okta OIDC App with an EKS cluster.

## Setting up OIDC with EKS + Okta

**Prerequisites:** Create an Okta OIDC Application with the Login redirect URI set to `http://localhost:8080`. Note down the OIDC Issuer URL and Client ID.

**Associate the OIDC provider with your EKS cluster:**

1. Login to AWS Console → EKS
2. Click on your cluster → Configuration → Authentication
3. Click **Associate a OIDC Provider** and fill in the Issuer URL and Client ID

**Test authentication using kubelogin:**

```bash
kubectl oidc-login setup \
  --oidc-issuer-url https://my.okta.com \
  --oidc-client-id <client-id> \
  --oidc-client-secret=<client-secret> \
  --oidc-extra-scope groups,email
```

This opens a browser for Okta login. On success, you get back a token with claims including email, groups, etc.

**Set up kubeconfig:**

```bash
kubectl config set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://my.okta.com \
  --exec-arg=--oidc-client-id=<client-id> \
  --exec-arg=--oidc-client-secret=<secret> \
  --exec-arg=--oidc-extra-scope=groups \
  --exec-arg=--oidc-extra-scope=email
```

**Verify cluster access:**

```bash
kubectl --user=oidc get nodes
kubectl config set-context --current --user=oidc
```

This works perfectly for EKS — once you configure the clusterrolebinding for the Okta group or user, developers can authenticate without using AWS principals at all.

## The GKE Problem

GKE doesn't support external OIDC providers yet, which means this is only a partial solution. I came across [this presentation](https://www.youtube.com/watch?v=6i7m54z3nqM) from Josh Van Leeuwen about `kube-oidc-proxy` which solves this gap.

To be continued in Part 2...

---

*Originally published on [Medium](https://muralidkt.medium.com/sso-for-managed-kubernetes-f85b4206d372) — May 14, 2021*
