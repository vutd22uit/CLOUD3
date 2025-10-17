
# Capstone Project Outline

## Đề tài 13: Xây dựng CI/CD pipeline bảo mật (DevSecOps) — tích hợp SAST, DAST, IaC scanning, container image scanning (Trivy/Anchore)

**Mô tả ngắn:**  
Xây dựng một CI/CD pipeline thực tế cho ứng dụng mẫu (web service + IaC) với tích hợp kiểm thử bảo mật toàn diện: SAST (code scanning), DAST (runtime scan), IaC scanning (Checkov/tfsec), container image scanning (Trivy/Anchore), SBOM generation/verification, ký image, và policy‑gating. Pipeline phải tự động hóa (pre‑commit → PR → build → test → deploy), có báo cáo, triage flow và tích hợp DevSecOps (shift‑left).

---

### 1. Đặt vấn đề (Problem Statement)

- Sai sót bảo mật dễ chui vào production qua CI/CD: secrets trong code, vuln trong dependency, misconfig IaC, insecure container images.  
- Các tổ chức cần pipeline tự động block các thay đổi kém an toàn, đồng thời giữ trải nghiệm developer (fast feedback).  
- Mục tiêu: thực hiện end‑to‑end DevSecOps pipeline reproducible cho lab/course, hướng tới nghề Cloud Security Engineer / DevSecOps.

---

### 2. Mục tiêu dự án (Objectives)

- Thiết kế repo mẫu chứa: ứng dụng (e.g., simple web app), Dockerfile, Terraform/Ansible IaC.  
- Triển khai CI pipeline (GitHub Actions / GitLab CI / Jenkins) tích hợp: pre‑commit checks, SAST (semgrep/sonar), dependency scan (Dependabot/Snyk), IaC scan (Checkov/tfsec), container build, image scan (Trivy/Anchore), SBOM (syft), image signing (cosign), DAST (OWASP ZAP) trên staging.  
- Định nghĩa gating policy: threshold vulnerabilities, block PR merge, auto PR fix for IaC where safe.  
- Tích hợp triage: create ticket (JIRA) hoặc create issue with remediation hints.  
- Đánh giá: pipeline latency, false positive rate, developer friction, security improvement (pre-prod vuln reduction).

---

### 3. Kiến trúc & Công nghệ sử dụng (Architecture & Tech Stack)

- **VCS / CI:** GitHub + GitHub Actions (recommended) or GitLab CI / Jenkins.  
- **App stack:** sample web app (Node/Python/Go) + Dockerfile + Terraform.  
- **SAST:** semgrep, Bandit (Python), ESLint (JS), Gosec (Go), SonarQube/SonarCloud (optional).  
- **Dependency scanning:** Dependabot (GitHub), Snyk, GitHub Code Scanning.  
- **IaC scanning:** Checkov, tfsec, Conftest/OPA Rego for policies.  
- **Container scanning:** Trivy (quick), Anchore Engine / Clair (policy/gates).  
- **SBOM & signing:** Syft (generate SBOM), Cosign / Notary v2 (sign & verify images).  
- **DAST:** OWASP ZAP (automation mode) or w3af.  
- **Policy & Admission:** OPA Gatekeeper (Kubernetes admission), Notary verification in deployment.  
- **Registry:** GitHub Container Registry / Harbor / Docker Registry with vulnerability scanning.  
- **Ticketing / ChatOps:** JIRA / GitHub Issues, Slack for notifications.  
- **IaC provisioning:** Terraform for infra, Ansible for config.  
- **Secrets management:** HashiCorp Vault / GitHub Secrets / Azure Key Vault.  
- **Observability:** ELK / Grafana for pipeline metrics & vulnerability dashboards.  

---

### 4. Threat Model & Use Cases (Security Controls Covered)

- **T1 — Sensitive secrets checked into repo** (API keys, DB passwords). Detect via pre‑commit (gitleaks) and fail build.  
- **T2 — Vulnerable dependency introduced** (high CVE). Block merge if severity threshold exceeded.  
- **T3 — IaC misconfiguration merged** (public storage, open SG). Block merge or create auto-fix PR.  
- **T4 — Insecure container image published** (unpatched base). Prevent push to prod registry.  
- **T5 — Runtime exploit discovered in staging:** DAST finds remote RCE → fail promotion to prod.  

---

### 5. Thiết kế Pipeline (Stages & Checks)

**Suggested stages:** Pre-commit → PR (CI) → Build → Image Scan → SBOM & Sign → Push to Registry (staging) → DAST in staging → Policy Gate → Deploy to Prod (if gated).

1. **Pre‑commit hooks (local)**
   - `pre-commit` config: run `semgrep`, `gitleaks`, `tfsec` for changed files, linters. Provide quick feedback to devs.  

2. **PR / CI checks (on push / PR)**
   - Checkout code; install deps.  
   - Run SAST (semgrep rules + language-specific analyzers).  
   - Run dependency scan (e.g., `npm audit`, `pip-audit`, Snyk test).  
   - Run IaC static scan (Checkov/tfsec) against Terraform/Ansible.  
   - Run unit tests & code quality metrics.  
   - Report results as GitHub Checks with annotations (files/lines). Block merge on failure for critical checks.

3. **Build & Container image stage**
   - Build image (Docker BuildKit). Tag with commit SHA.  
   - Generate SBOM: `syft <image> -o json > sbom.json`.  
   - Scan image with Trivy/Anchore: if vulnerabilities > threshold, fail.  
   - Sign image with cosign: `cosign sign --key ... ghcr.io/org/image:sha`.  
   - Push signed image to registry (only if signed & scan OK).

4. **Staging & DAST**
   - Deploy to isolated staging namespace via Terraform/Kubernetes manifests.  
   - Run OWASP ZAP scan automation against staging endpoint → gather findings.  
   - If DAST finds critical issues, fail promotion and create issue with context.  

5. **Policy Gate (pre-prod → prod)**
   - Verify SBOM, signature, and registry policy (admission webhook to check cosign signature + policy via OPA Gatekeeper).  
   - Enforce runtime policies (no privileged containers, read-only filesystem, drop capabilities) via PodSecurity and OPA.  
   - If all gates pass, promote deployment to production.

---

### 6. Sample CI (GitHub Actions) snippet

```yaml
name: CI-Secure
on: [push, pull_request]
jobs:
  prechecks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
      - name: Run gitleaks
        uses: zricethezav/gitleaks-action@v1
      - name: Run Checkov (IaC)
        uses: bridgecrewio/checkov-action@v10
  build_and_scan:
    needs: prechecks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build container
        run: docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
      - name: Generate SBOM with Syft
        run: syft ghcr.io/${{ github.repository }}:${{ github.sha }} -o json > sbom.json
      - name: Scan image with Trivy
        run: trivy image --severity CRITICAL,HIGH --exit-code 1 ghcr.io/${{ github.repository }}:${{ github.sha }}
      - name: Sign image with Cosign
        run: cosign sign --key ${{ secrets.COSIGN_KEY }} ghcr.io/${{ github.repository }}:${{ github.sha }}
      - name: Push to GHCR
        run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

(Extend with caching, buildx, and artifact upload as needed.)

---

### 7. IaC & Configuration Policy (OPA / Conftest / Checkov)

- Use Checkov/tfsec in CI for Terraform. Use Conftest with Rego to implement org‑specific rules (e.g., disallow public S3, require tags, restrict instance types).  
- Example Rego snippet (deny public S3 in terraform plan):

```rego
package terraform.s3
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.after.acl == "public-read"
  msg = sprintf("S3 %s is public", [resource.address])
}
```

- Use Gatekeeper on K8s to block deployments that violate PodSecurity / OPA policies.

---

### 8. Container Security (Scanning, Signing, Runtime)

- **Static scanning:** Trivy CLI during CI to detect CVEs in OS packages and application dependencies.  
- **Policy scanning:** Anchore can apply policy bundles and provide detailed reports.  
- **SBOM:** syft to create SBOM (SPDX/CycloneDX), store SBOM artifact in registry or artifact store. SBOM used for vulnerability triage & SCA.  
- **Image signing & verification:** cosign for supply chain signing; verify signature in admission webhook before runtime.  
- **Runtime protections:** Falco for runtime detection, image vulnerability blocking via admission controller, PodSecurity.  
- **Registry hardening:** use Harbor or GHCR with vulnerability scanning and immutable tags for prod images.

---

### 9. DAST Integration (OWASP ZAP)

- Automate ZAP in CI or post‑deploy stage: use ZAP baseline & active scan against staging URL.  
- Parse ZAP report (JSON) and convert to GitHub annotations or create issue for critical findings.  
- For authenticated scans, provision test accounts / test tokens with limited privileges.  
- Limit DAST scope to non‑destructive tests and ensure staging is isolated.

---

### 10. Triage, Issue Creation & Remediation Automation

- On failed checks, create annotated GitHub Check runs + COMMENTS on PR with details and remediation hints (e.g., semgrep rule, vulnerable dependency upgrade path).  
- Critical failures automatically create JIRA issue (with trace, SBOM) and assign to owner.  
- For IaC issues, optionally create auto-fix PR (use Bridgecrew/Checkov auto-fix or scripted terraform patch) — require manual review.  
- Maintain suppression/ALLOWLIST process: require documented exception ticket with TTL.

---

### 11. Metrics & Evaluation (KPIs)

- **Security metrics:** number of critical/high vuln prevented from reaching prod; average vulnerabilities per image over time; % PRs blocked due to security checks.  
- **Operational metrics:** pipeline latency (time from PR → build → gated), false-positive rate (FP per check), time-to-fix (median), developer satisfaction (survey).  
- **Compliance metrics:** SBOM coverage %, images signed %, IaC compliance % (CIS/Terraform checks).  
- **Quality metrics:** mobile test coverage, unit tests passing rate.

---

### 12. Testing & Validation (Test Plan)

- Unit tests & integration tests for app.  
- Simulate injection of secrets & ensure gitleaks detects.  
- Add synthetic vulnerable dependency to check enforcement (e.g., pin a package with known CVE).  
- Inject IaC misconfig and observe Checkov blocking PR.  
- Build image with known CVE base and ensure Trivy fails the CI.  
- Run DAST against staging with a test RCE endpoint to verify detection.  
- Validate admission webhook denies unsigned/unverified images.

---

### 13. Security & Privacy Considerations

- Do NOT store production secrets in CI logs; use secret masking and secure storage for keys.  
- Run DAST only in isolated staging; avoid attacking external services.  
- Ensure SBOMs that include PII are sanitized before sharing.  
- Manage cosign keys in secure vault (not in plain secrets).

---

### 14. Deliverables (nộp cuối)

- GitHub repo: sample app, Terraform/Ansible, pipeline workflows (GitHub Actions), pre-commit config.  
- Scripts & Dockerfiles: image build & SBOM generation scripts.  
- CI run artifacts: sample reports from semgrep, Checkov, Trivy, ZAP.  
- Policy definitions: OPA Rego / Checkov policies, Gatekeeper manifests.  
- Demo video (5–10 phút) showing pipeline run, blocking & remediation.  
- Technical report (15–25 trang) with evaluation metrics, developer impact analysis, lessons learned.  
- Playbooks: triage flow, false-positive handling, exception process.

---

### 15. Rubric đánh giá (Suggested Grading Rubric)

- **Pipeline completeness (25%)**: pre-commit → CI → build → scan → deploy stages implemented.  
- **Security coverage (25%)**: SAST + IaC + Container + DAST integrated and demonstrated.  
- **Automation & triage (20%)**: issue creation, auto-fix PRs or remediation playbooks, suppression process.  
- **Testing & metrics (15%)**: test plan executed, metrics collected & analyzed.  
- **Documentation & reproducibility (15%)**: README, runbook, scripts, demo.

---

### 16. Milestones & Timeline (14 tuần đề xuất)

- Tuần 1–2: Project planning, choose sample app & infra, baseline CI.  
- Tuần 3–4: Implement pre-commit hooks (semgrep, gitleaks, tfsec).  
- Tuần 5–6: Integrate SAST & dependency scanning in CI.  
- Tuần 7–8: Implement container build, SBOM generation, Trivy/Anchore scanning.  
- Tuần 9: Image signing with cosign & push to registry.  
- Tuần 10: Deploy staging & integrate OWASP ZAP DAST.  
- Tuần 11: Set up admission policies (OPA Gatekeeper) & verify gating.  
- Tuần 12: Automation: issue creation & triage flows, auto-fix PR for IaC.  
- Tuần 13: End-to-end testing, metrics collection.  
- Tuần 14: Finalize report, demo and presentation.

---

### 17. Example commands & snippets (Appendix)

**Run semgrep locally:**  

```bash
semgrep --config=auto
```

**Run Checkov on Terraform:**  

```bash
checkov -d ./terraform --skip-check CKV_AWS_20
```

**Generate SBOM with syft:**  

```bash
syft ghcr.io/org/image:tag -o spdx-json > sbom.spdx.json
```

**Trivy scan image:**  

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 ghcr.io/org/image:tag
```

**Sign image with cosign:**  

```bash
cosign sign --key cosign.key ghcr.io/org/image:tag
# verify
cosign verify --key cosign.pub ghcr.io/org/image:tag
```

**Run OWASP ZAP baseline scan (Docker):**  

```bash
docker run --rm -v $(pwd):/zap/wrk/:Z owasp/zap2docker-stable zap-baseline.py -t https://staging.example.com -r zap_report.html
```

---

### 18. Extensions & Advanced Ideas (Optional)

- Integrate in-toto for supply chain provenance & SLSA levels.  
- Use graph DB to correlate SBOM components to vuln databases and produce impact analysis.  
- Add automated patching pipelines for low‑risk vulns (create PR, run tests, merge with approvals).  
- Implement feature flags to decouple security gating for fast rollback.  
- Explore reproducible builds and reproducible SBOMs.

---

### 19. Ethical & Practical Notes for Instructors

- Provide sandbox registry & test accounts; rotate test keys after course.  
- Enforce non-destructive DAST scope & preapproval for test endpoints.  
- Encourage reproducibility: pin versions and containerize tools.
