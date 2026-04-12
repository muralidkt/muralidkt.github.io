---
title: "Building Multi-Architecture Container Images Using Kaniko and GitLab CI"
author: Murali Kotharamban
pubDatetime: 2023-12-15T00:00:00.000Z
slug: multi-arch-container-kaniko-gitlab
featured: false
draft: false
tags:
  - Kubernetes
  - Docker
  - GitLab
  - AWS
  - Graviton
  - DevOps
  - Platform Engineering
description: "How to build multi-architecture container images using Kaniko and GitLab CI, targeting both amd64 and arm64 for AWS Graviton on ECS Fargate — with native runners per architecture and manifest-tool to combine them."
---

In the ever-evolving landscape of cloud computing, optimising performance and cost-efficiency has always been a key goal for developers, DevOps, and Platform Engineers. My journey into multi-architecture container images began with exactly that aim.

The advent of **AWS Graviton processors** presented an opportunity to enhance performance and reduce cost for container workloads running on AWS ECS Fargate. Graviton is ARM64-based — which means traditional container images built for x86-64/amd64 won't run on it natively. To leverage Graviton without maintaining separate pipelines per architecture, the answer is **multi-architecture container images**: a single image reference that transparently serves the right layers to the right host.

---

## Why Kaniko?

In our setup, GitLab CI runs inside a Kubernetes cluster. The traditional approach — mounting the Docker socket into a CI container — is a significant security risk: it gives the CI job root-equivalent access to the host. Kaniko solves this by building container images entirely in userspace, without a Docker daemon, making it safe to run inside Kubernetes pods.

It's also straightforward to integrate with ECR: Kaniko handles AWS credential resolution via the standard SDK chain, so if your GitLab runner pod has an IAM role, authentication just works.

---

## The Build Strategy: Native Runners Per Architecture

The approach is to use **dedicated GitLab runners per architecture** — one tagged `runner-amd64`, one tagged `runner-arm64`. Each runner builds its own native image. This is more reliable than cross-compilation (using QEMU emulation) because:

- Builds run on native hardware — no emulation overhead or subtle compatibility issues
- Dependencies that include native binaries or compile C extensions just work
- Build times are significantly faster

The pipeline uses a shared `.kaniko-build` job template, with architecture-specific jobs extending it:

```yaml
.kaniko-build:
  stage: build-image
  variables:
    AWS_DEFAULT_REGION: eu-central-1
    AWS_ACCOUNT_ID: "123456789012"
  image:
    name: gcr.io/kaniko-project/executor:v1.18.0-debug
  script:
    - |
      /kaniko/executor \
        --context . \
        --dockerfile Dockerfile \
        --destination "${AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com/image:${ARCH}-${CI_COMMIT_SHORT_SHA}"

build-amd64:
  extends: .kaniko-build
  variables:
    ARCH: amd64
  tags:
    - runner-amd64

build-arm64:
  extends: .kaniko-build
  variables:
    ARCH: arm64
  tags:
    - runner-arm64
```

Each image is tagged with both the architecture and the Git commit SHA — for example:

```
123456789012.dkr.ecr.eu-central-1.amazonaws.com/image:amd64-a1b2c3d4
123456789012.dkr.ecr.eu-central-1.amazonaws.com/image:arm64-a1b2c3d4
```

This makes it easy to pull and test a specific architecture image during debugging, and keeps the registry organised.

---

## How Multi-Architecture Images Actually Work

A multi-architecture image isn't a special image format — it's a **manifest list** (also called an OCI image index). The manifest list is a metadata document stored in the registry that maps platform identifiers (`linux/amd64`, `linux/arm64`) to the digest of the corresponding single-architecture image.

When a container runtime pulls an image, it sends its platform information to the registry. The registry returns the right manifest for that platform. To the user, it's completely transparent — the same `docker pull` command works on an x86 laptop and a Graviton Fargate task.

```
image:a1b2c3d4  (manifest list)
├── linux/amd64 → sha256:abc123...  (amd64 image layers)
└── linux/arm64 → sha256:def456...  (arm64 image layers)
```

---

## Creating the Manifest List with manifest-tool

Once both architecture images are built and pushed to ECR, the final step is combining them into a manifest list. We use **manifest-tool** for this — a purpose-built CLI for creating and pushing OCI manifest lists.

```yaml
.build-multi-arch-image:
  image:
    name: your-registry/manifest-tool:latest
  variables:
    AWS_DEFAULT_REGION: eu-central-1
    AWS_ACCOUNT_ID: "123456789012"
  script:
    - |
      manifest-tool push from-args \
        --platforms linux/amd64,linux/arm64 \
        --template "${AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com/image:ARCH-${CI_COMMIT_SHORT_SHA}" \
        --target "${AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com/image:${CI_COMMIT_SHORT_SHA}"

build-multi-arch-image:
  stage: build-multi-arch-image
  extends: .build-multi-arch-image
  tags:
    - runner-amd64
```

The `--template` flag uses `ARCH` as a literal placeholder that manifest-tool substitutes with `amd64` and `arm64` when locating the individual architecture images. The `--target` is the final multi-arch tag — just the commit SHA, without an architecture prefix.

After this stage runs, pulling `image:a1b2c3d4` from any host will automatically get the right architecture.

---

## Full Pipeline Stages

```
build-image (parallel)
  ├── build-amd64   [runner-amd64]
  └── build-arm64   [runner-arm64]

build-multi-arch-image
  └── build-multi-arch-image  [runner-amd64]  ← depends on both build jobs
```

The multi-arch manifest job must run after both architecture builds complete — GitLab's `needs` keyword handles this dependency explicitly.

---

## ECR Considerations

A few things worth knowing when using ECR with multi-arch images:

- ECR supports OCI manifest lists natively — no special configuration needed
- The manifest list and all referenced image digests must be in the **same repository**
- ECR authentication tokens expire after 12 hours — for long pipelines, you may need to re-authenticate between stages. Kaniko handles this via the AWS SDK; manifest-tool needs the token passed explicitly or via a credential helper
- Enable **immutable tags** in ECR to prevent accidental overwrites of commit-SHA-tagged images

---

## Results on AWS Graviton (ECS Fargate)

After rolling this out, workloads running on ARM64 Fargate tasks with Graviton processors showed:

- **~20% cost reduction** compared to equivalent x86 task sizes (Graviton pricing is lower)
- **Comparable or better performance** for Node.js services — V8 on ARM64 performs well
- Zero changes required to application code — the multi-arch image handles everything transparently

The pipeline overhead — two parallel builds plus a manifest step — added roughly 2-3 minutes to total pipeline time, negligible given the savings.

---

## Summary

Multi-architecture container images are the right approach for teams targeting both x86 and ARM infrastructure. The combination of:

- **Kaniko** for daemonless, secure builds inside Kubernetes
- **Dedicated native runners** per architecture for reliable, fast builds
- **manifest-tool** to combine images into an OCI manifest list

...gives you a clean, maintainable pipeline that works transparently across architectures. If you're running on AWS and haven't evaluated Graviton yet, it's worth a look — the performance-to-cost ratio is hard to beat for most containerised workloads.

---

*Murali Kotharamban is a Senior Platform Engineer at Quantagonia GmbH, Munich, building cloud-native platforms and AI infrastructure on AWS.*
