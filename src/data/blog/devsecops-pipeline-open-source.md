---
title: "Building a DevSecOps Pipeline Using Open Source Tools"
author: Murali Kotharamban
pubDatetime: 2022-03-08T00:00:00.000Z
slug: devsecops-pipeline-open-source
featured: false
draft: false
tags:
  - DevSecOps
  - Security
  - CI/CD
  - GitLab
  - Open Source
  - DevOps
description: "A complete DevSecOps pipeline using open source tools — Dependency-Track, CodeQL, Gitleaks, Checkov, Trivy, and DefectDojo — integrated into GitLab CI/CD for a Terraform + Node.js + GCP Cloud Run stack."
---

Security is most effective when it's built into the development process — not bolted on at the end. A DevSecOps pipeline shifts security left, giving developers immediate feedback on vulnerabilities before code ever reaches production.

In this post, I'll walk through building a complete DevSecOps pipeline using open source tools, integrated into a GitLab CI/CD pipeline. I'll cover the same approach for GitHub Actions and Jenkins in follow-up posts.

---

## The Stack

The tools you choose depend on your language, platform, and threat model. For a generic CI/CD pipeline covering the most common attack surfaces, this is the stack I'd reach for:

| Category | Tool |
|----------|------|
| Software Composition Analysis (SCA) | OWASP Dependency-Track |
| Static Application Security Testing (SAST) | CodeQL |
| Secret Scanning | Gitleaks |
| Infrastructure as Code Scanning | Checkov |
| Container Scanning | Trivy |
| Vulnerability Management | DefectDojo |

For the demo, I'm using a repository that deploys a sample Node.js microservice to **GCP Cloud Run** using Terraform as IaC. Cloud Run is Google's serverless container platform — it handles orchestration so you don't manage infrastructure directly.

---

## Why These Tools?

**OWASP Dependency-Track** tracks your third-party dependencies against known CVEs (NVD, OSS Index, GitHub Advisories). It's not just a point-in-time scan — it continuously monitors your component inventory and alerts you when new vulnerabilities are disclosed for packages you already depend on.

**CodeQL** is GitHub's semantic code analysis engine, open sourced via GitHub Actions. Unlike pattern-matching SAST tools, CodeQL builds a queryable model of your code and finds vulnerabilities based on data flow and control flow. It catches SQL injection, XSS, path traversal, and many other vulnerability classes that simpler tools miss.

**Gitleaks** scans your Git history and working tree for secrets — API keys, tokens, credentials, private keys. The history scan is important: a secret committed and immediately deleted is still in your git history and potentially exposed.

**Checkov** is Bridgecrew's IaC scanner. It checks Terraform, CloudFormation, Kubernetes manifests, Dockerfiles, and ARM templates against hundreds of security and compliance rules. For a Terraform + GCP setup, it'll catch things like Cloud Run services with unauthenticated access, GCS buckets with public ACLs, and missing VPC Service Controls.

**Trivy** scans container images for OS-level and application-level vulnerabilities. It covers the full image — base OS packages, language runtimes, and application dependencies. Fast, accurate, and works well in CI with its table and JSON output modes.

**DefectDojo** is the glue. It's an OWASP vulnerability management platform that ingests scan results from all of the above tools, deduplicates findings, tracks remediation status, and gives your team a single pane of glass for security posture. It has native importers for Dependency-Track, CodeQL SARIF, Gitleaks JSON, Checkov JSON, and Trivy JSON — no custom parsing needed.

---

## The Pipeline Structure

The GitLab CI/CD pipeline covers three core workflows:

1. Deploy Terraform infrastructure for Cloud Run
2. Build the Docker image
3. Deploy the container to Cloud Run

Security scans are inserted at the appropriate stages — early enough to be useful, positioned so they don't block unrelated work:

```yaml
stages:
  - validate
  - scan-code
  - build
  - scan-container
  - deploy
  - report

variables:
  DEFECTDOJO_URL: "https://your-defectdojo-instance"
  DEFECTDOJO_TOKEN: $DEFECTDOJO_API_TOKEN  # stored as CI/CD secret
  PRODUCT_NAME: "nodejs-cloudrun-demo"

# ─── Secret Scanning ─────────────────────────────────────────────────────────
gitleaks:
  stage: validate
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --source . --report-format json --report-path gitleaks-report.json
  artifacts:
    paths: [gitleaks-report.json]
    when: always
  allow_failure: true

# ─── IaC Scanning ────────────────────────────────────────────────────────────
checkov:
  stage: validate
  image: bridgecrew/checkov:latest
  script:
    - checkov -d ./terraform --output json > checkov-report.json || true
  artifacts:
    paths: [checkov-report.json]
    when: always

# ─── SAST ────────────────────────────────────────────────────────────────────
codeql:
  stage: scan-code
  image: github/codeql-action/analyze:latest
  script:
    - codeql database create codeql-db --language=javascript
    - codeql analyze codeql-db javascript-security-extended.qls --format=sarif-latest --output=codeql-report.sarif
  artifacts:
    paths: [codeql-report.sarif]
    when: always

# ─── SCA ─────────────────────────────────────────────────────────────────────
dependency-track:
  stage: scan-code
  image: node:18-alpine
  script:
    - npm ci
    - npm install -g @cyclonedx/cyclonedx-npm
    - cyclonedx-npm --output-file bom.json
    - |
      curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
        -H "Authorization: Token $DEFECTDOJO_TOKEN" \
        -F "scan_type=Dependency Track Finding Packaging Format (DTRACK) Scan" \
        -F "file=@bom.json" \
        -F "product_name=$PRODUCT_NAME" \
        -F "engagement_name=CI Pipeline"
  artifacts:
    paths: [bom.json]
    when: always

# ─── Container Build ─────────────────────────────────────────────────────────
docker-build:
  stage: build
  image: docker:latest
  services: [docker:dind]
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# ─── Container Scanning ──────────────────────────────────────────────────────
trivy:
  stage: scan-container
  image: aquasec/trivy:latest
  script:
    - trivy image --format json --output trivy-report.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    paths: [trivy-report.json]
    when: always

# ─── Upload All Results to DefectDojo ────────────────────────────────────────
upload-to-defectdojo:
  stage: report
  image: python:3.11-slim
  script:
    - pip install requests
    - |
      python3 << 'EOF'
      import requests, os

      base = os.environ["DEFECTDOJO_URL"]
      token = os.environ["DEFECTDOJO_TOKEN"]
      headers = {"Authorization": f"Token {token}"}
      product = os.environ["PRODUCT_NAME"]

      scans = [
          ("gitleaks-report.json",  "Gitleaks Scan"),
          ("checkov-report.json",   "Checkov Scan"),
          ("codeql-report.sarif",   "SARIF"),
          ("trivy-report.json",     "Trivy Scan"),
      ]

      for filename, scan_type in scans:
          if not os.path.exists(filename):
              print(f"Skipping {filename} — not found")
              continue
          with open(filename, "rb") as f:
              r = requests.post(
                  f"{base}/api/v2/import-scan/",
                  headers=headers,
                  data={"scan_type": scan_type, "product_name": product, "engagement_name": "CI Pipeline"},
                  files={"file": f},
              )
          print(f"{filename}: {r.status_code}")
      EOF
  dependencies:
    - gitleaks
    - checkov
    - codeql
    - trivy
  when: always
```

---

## Where to Place Scans in the Pipeline

Placement matters:

- **Gitleaks and Checkov** run at `validate` — before any build happens. No point building an image if you've already committed AWS credentials or your Terraform has public buckets.
- **CodeQL and Dependency-Track** run at `scan-code` — against source, in parallel with each other.
- **Trivy** runs at `scan-container` — must run after the image is built, so it catches vulnerabilities introduced by the base image or build process.
- **DefectDojo upload** runs last, `when: always` — even if a scan fails or a stage is skipped, upload whatever results exist.

---

## DefectDojo Setup

I'm running DefectDojo on **AWS ECS Fargate** — containers without managing EC2, with Fargate handling scaling. You can equally run it on Kubernetes with the official Helm chart, or locally with `docker-compose` for evaluation.

Once running, DefectDojo gives you:

- A product/engagement model that maps to your repos and sprint cycles
- Deduplication across scanners — the same CVE from Trivy and Dependency-Track won't create two separate findings
- Remediation tracking — assign findings to developers, track SLA compliance
- **SAML/OAuth SSO and RBAC** — integrate with your existing IAM solution and give developers direct access

That last point matters. DefectDojo should not be a management reporting tool. It should be where your developers check their open findings, mark false positives, and track what they've fixed. If developers can't see or act on the results, the pipeline adds friction without adding value.

---

## Commercial Alternatives

Worth mentioning: there are excellent commercial platforms — Snyk, Veracode, Checkmarx, Prisma Cloud — that cover this entire spectrum with tighter CI/CD integration, managed rule updates, and enterprise support. If your organisation has the budget and needs low-maintenance operation, they're worth evaluating.

The open source stack works well if you have the engineering capacity to operate DefectDojo and tune your rule sets. The main cost is operational overhead, not licensing.

---

## The Bigger Picture

Technology is only one third of DevSecOps. The other two thirds are **people** and **process**.

A pipeline that surfaces 300 findings and dumps them on developers with no context, no prioritisation, and no ownership model will be ignored within a month. The tooling needs to be backed by:

- Developer security training — they need to understand what findings mean
- Clear ownership — who fixes what, and by when
- Triage process — distinguishing critical from noise
- Security champions in each team who drive secure coding practices

The goal is a culture where security is a shared responsibility, not a gate at the end of the process. The pipeline is the mechanism. The culture is what makes it actually work.

A successful DevSecOps program is a combination of Technology, People, and Processes — and equal if not more importance should be given to building that culture through awareness and process than to the tooling itself.

---

## What's Next

In follow-up posts, I'll cover the same pipeline implemented with **GitHub Actions** and **Jenkins** — the tool choices stay the same, the syntax changes.

---

*Murali Kotharamban is a Platform Engineer with over a decade of experience in cloud security and DevSecOps, building security programs across AWS and GCP.*
