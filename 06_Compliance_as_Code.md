
# Capstone Project Outline  

## Đề tài 6: Compliance‑as‑Code — xây dựng framework kiểm thử CIS Benchmark / ISO 27017 / PCI‑DSS cho OpenStack & AWS, tự động chặn policy vi phạm

**Mô tả ngắn:**  
Xây dựng một framework *Compliance‑as‑Code* để kiểm thử, báo cáo và tự động ngăn chặn (block/deny/remediate) các policy vi phạm theo chuẩn (CIS Benchmarks, ISO‑27017, PCI‑DSS) trên hai môi trường: OpenStack (private) và AWS (public). Framework bao gồm: mapping controls → checks, IaC pre‑deploy gate, runtime continuous scanning, automatic enforcement (Azure/AWS policy, Cloud Custodian, Ansible), và CI/CD integration để đảm bảo compliance liên tục.

---

### 1. Đặt vấn đề (Problem Statement)

- Nhiều tổ chức gặp khó trong việc đảm bảo tuân thủ liên tục (continuous compliance) khi đa dạng cloud và IaC. Manual audits tốn thời gian, dễ sai sót.  
- **Compliance‑as‑Code (CaC)** giúp chuyển các yêu cầu chuẩn mực (CIS/ISO/PCI) thành checks, tests và policies có thể chạy tự động, tích hợp vào pipeline để ngăn lỗi trước khi deploy.  
- Mục tiêu: xây dựng framework tái sử dụng, audit‑friendly, có khả năng *block* deployment hoặc *auto‑remediate* khi phát hiện vi phạm.

---

### 2. Mục tiêu dự án (Objectives)

- Lựa chọn và mapping subset các control từ: **CIS Benchmarks (AWS, Linux image), ISO‑27017 (Cloud controls), PCI‑DSS basics** cho scope lab.  
- Thiết kế pipeline CaC: IaC static checks (pre‑commit/CI), runtime checks (periodic scanning), enforcement (deny/modify/auto‑remediate).  
- Triển khai sample checks cho OpenStack và AWS, và tích hợp với CI/CD (GitHub Actions/GitLab CI).  
- Implement blocking gates: cfn-guard / cfn-nag / Checkov gate cho AWS CloudFormation/Terraform; OPA/Gatekeeper admission for K8s (if used); OpenStack policy checks via tests (Ansible + OpenSCAP).  
- Đánh giá hiệu quả: coverage of selected controls, false positives, MTTR, developer friction (lead time).

---

### 3. Kiến trúc & Công nghệ sử dụng (Architecture & Tech Stack)

- **Cloud platforms:** OpenStack (lab) + AWS (free tier).  
- **IaC:** Terraform (preferred) + optional CloudFormation.  
- **IaC Scanners / Pre‑deploy gates:** Checkov, tfsec, cfn-guard, cfn-nag, pre-commit hooks.  
- **Compliance tests / Runtime scanners:** Chef InSpec (InSpec profiles for CIS/PCI/ISO), OpenSCAP (for Linux images), Lynis (optional), ScoutSuite (cloud posture).  
- **Policy Engines / Enforcement:** OPA (Rego) + Gatekeeper (K8s), AWS Config Rules + remediation, AWS Organizations SCPs (preventive), Cloud Custodian (remediation).  
- **CI/CD:** GitHub Actions / GitLab CI to run tests & block PRs.  
- **Logging & Audit:** ELK stack or AWS CloudTrail + AWS Config Aggregator, S3 for evidence storage.  
- **Secrets & Key management:** HashiCorp Vault / AWS KMS for storing signing keys and credentials for test runners.  
- **Messaging / Ticketing:** Slack / Email / JIRA for notifications & approval flows.  

---

### 4. Scope controls & Mapping (Example mapping)

**Chọn subset controls để thực hiện trong project (ví dụ):**  

- **CIS AWS Foundations**: Ensure S3 buckets are not public; Ensure CloudTrail enabled & logs encrypted; Ensure MFA enabled on root account.  
- **CIS Linux / Cloud images:** Ensure SELinux/AppArmor enabled; Ensure SSH root login disabled.  
- **ISO‑27017 (Cloud specific):** Logical separation between customer environments; Protection of data in transit & at rest (encryption).  
- **PCI‑DSS (basics for cloud):** Cardholder data encryption at rest; Restrict access to cardholder data by business need‑to‑know; Logging/monitoring retention.  

**Mapping:** Create a mapping table: `control_id -> check_type -> IaC check (Checkov) -> runtime check (InSpec/OpenSCAP) -> enforcement mechanism (deny/auto-remediate)`.

---

### 5. Threat Model & Failure Modes (Use Cases)

- **F1 — IaC misconfig deployed to prod**: pre‑deploy checks missed or bypassed.  
- **F2 — Drift between IaC and runtime**: operator manual change causes non‑compliant state.  
- **F3 — Developer circumvents checks (force merge)**: lack of enforcement leads to non‑compliance.  
- **F4 — False positive blocking critical deploys**: must balance safety vs availability.  
- **F5 — Evidence tampering**: audit logs not hardened.  

---

### 6. Design & Implementation (Step‑by‑step)

**High-level flow:**  

1. **Define control set & profiles:** pick a realistic subset from CIS/ISO/PCI for the course scope.  
2. **Author test suites:** write Chef InSpec profiles for runtime checks and Checkov policies/OPA Rego for IaC checks.  
3. **CI integration:** configure GitHub Actions to run Checkov/tfsec/InSpec on PRs; fail PR on critical controls.  
4. **Runtime scanning:** schedule ScoutSuite/CloudSploit & InSpec periodic scans; results pushed to SIEM/Elastic.  
5. **Enforcement & remediation:** configure AWS Config rules with automatic remediation (SSM automation or Lambda) & Cloud Custodian policies for remediation; for OpenStack use Ansible playbooks to revert non‑compliant settings.  
6. **Evidence & reporting:** store scan reports, InSpec results, remediation logs in S3/ELK for auditors.  
7. **Governance UI:** simple dashboard to show compliance score, trends, open violations, remediation status.  

**Implementation notes per component:**  

- **IaC pre‑deploy gate:** Use Checkov + cfn-guard in GitHub Actions. Example: `checkov -d . --soft-fail=false` and block merge on fail.  
- **Policy as code (OPA):** write Rego policies to enforce e.g., no public S3 in Terraform plan via Conftest.  
- **Runtime tests with InSpec:** create profiles that test actual deployed resources (e.g., `aws_s3_bucket(bucket_name)` InSpec resource).  
- **Remediation:** Cloud Custodian policies for S3 public fix; Lambda/Ansible playbook to change OpenStack security groups.  

---

### 7. Data & Evidence Handling (Logs, Reports)

- **Inputs:** IaC plan diffs, scanner reports (JSON), cloud provider APIs, InSpec JSON output.  
- **Normalization:** canonical schema `{control_id, resource_id, provider, found_at, severity, evidence, remediation_status}`.  
- **Storage & Retention:** store immutable evidence with versioned S3 bucket + object lock (if available) or ELK index with retention policy.  
- **Audit trails:** every remediation must log requestor, trigger (auto/manual), before/after state, and signature for compliance evidence.

---

### 8. Policy Enforcement Modes (Block vs Remediate)

- **Preventive (Block at IaC/CI):** block PR/merge if Checkov/InSpec failing critical controls. Use cfn-guard/cfn-nag for CloudFormation.  
- **Preventive (Org level):** AWS Organizations SCPs / Service Control Policies to block risky operations.  
- **Admission control (K8s):** OPA Gatekeeper to deny non‑compliant manifests.  
- **Corrective (Runtime):** Cloud Custodian or AWS Config automated remediation (runbook via SSM or Lambda).  
- **Advisory:** raise Jira ticket for manual remediation for sensitive findings.  

---

### 9. Sample Policy Snippets (Examples)

**A. Conftest / Rego example (deny public S3 in Terraform plan):**

```rego
package terraform.s3

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  bucket := resource.change.after
  bucket.acl == "public-read"
  msg = sprintf("S3 bucket %s has public-read ACL", [resource.address])
}
```

**B. AWS Config Rule (managed rule) example:** Use managed rule `s3-bucket-public-read-prohibited` or custom rule to detect public access.  

**C. Chef InSpec example (aws_s3_bucket):**

```ruby
control 's3-1.1' do
  impact 1.0
  title 'S3 buckets should not be publicly accessible'
  describe aws_s3_bucket(bucket_name: attribute('bucket_name')) do
    it { should_not be_public }
  end
end
```

**D. Cloud Custodian policy to remediate public buckets:**

```yaml
policies:
  - name: s3-enforce-block-public
    resource: aws.s3
    filters:
      - "PublicAccessBlockConfiguration.BlockPublicPolicy": false
    actions:
      - type: set-public-access-block
        BlockPublicPolicy: true
        BlockPublicAcls: true
```

---

### 10. Testing & Validation (Test Plan)

**Unit tests:** validate Rego policies with sample IaC JSON inputs; run InSpec unit tests against mocked resource responses.  
**Integration tests:** pipeline on feature branch: terraform plan + conftest/checkov → block if fail. Deploy to sandbox env → run InSpec/ScoutSuite → verify remediation triggers.  
**Compliance acceptance tests:** define acceptance criteria for each control (pass/fail logic).  
**Performance tests:** time to run full suite for X resources; ensure CI runtime acceptable.  
**False positive handling:** create mechanism for exceptions with justifications (exception ticket, TTL).

---

### 11. Evaluation Metrics (KPIs)

- **Coverage:** % of selected controls implemented as automated checks.  
- **Prevention rate:** % of prevented non‑compliant PRs/deploys.  
- **Remediation rate:** % of runtime violations remediated automatically.  
- **False positive rate (FPR):** % of alerts marked as false.  
- **Compliance score:** composite score across controls (e.g., number passed / total).  
- **Developer friction:** average time to merge PRs (delta before/after).

---

### 12. Grading Rubric (Suggested)

- **Control mapping & test authoring (25%)**: clear mapping, InSpec & Rego/Checkov rules implemented.  
- **CI/CD integration & blocking gates (20%)**: PR gating implemented + tests pass/fail demonstrable.  
- **Runtime scanning & remediation (25%)**: periodic scanning, Cloud Custodian/Ansible remediations working.  
- **Evidence & reporting (15%)**: reports stored, dashboard & audit evidence.  
- **Testing & docs (15%)**: test cases, user guide, and final report.

---

### 13. Milestones & Timeline (14 tuần đề xuất)

- Tuần 1–2: Lựa chọn subset controls & mapping, chuẩn bị lab OpenStack + AWS sandbox.  
- Tuần 3–4: Implement IaC pre‑deploy checks (Checkov/Conftest), integrate in GitHub Actions.  
- Tuần 5–6: Write InSpec profiles & OpenSCAP templates for image/runtime checks.  
- Tuần 7–8: Implement runtime scanning pipeline (ScoutSuite/CloudSploit) & push results to ELK.  
- Tuần 9: Implement enforcement (Cloud Custodian, AWS Config remediations, Ansible for OpenStack).  
- Tuần 10: Implement exception handling & evidence storage.  
- Tuần 11: Integration tests & pilot run.  
- Tuần 12: Collect metrics: FP analysis & tune policies.  
- Tuần 13: Finalize dashboard & report.  
- Tuần 14: Presentation, demo, handover documentation.

---

### 14. Deliverables (nộp cuối)

- Git repo: IaC policies (Checkov/Conftest/Rego), InSpec profiles, CI configs.  
- Runtime pipeline code: scanner runners, normalizer, remediation scripts.  
- Dashboards: Kibana/ELK dashboards + sample reports.  
- Evidence bundle: archived scan & remediation logs for auditor.  
- Demo video (5–10 phút), Technical report (15–25 trang), Slides (10–15 slides).

---

### 15. Ethical, Legal & Privacy Considerations

- Xin phép và sử dụng sandbox accounts; không quét hoặc thay đổi tài nguyên bên ngoài phạm vi.  
- Lưu trữ logs có PII phải tuân thủ privacy rules (masking/anonymization).  
- Exception process must be auditable & time‑bounded to avoid compliance loopholes.

---

### 16. Extensions & Advanced Ideas (Optional)

- Integrate HashiCorp Sentinel for policy enforcement in Terraform Cloud / Enterprise.  
- Build a multi‑cloud aggregator for compliance scoreboard (AWS + Azure + GCP + OpenStack).  
- Add automated evidence generation for auditors (PDF exports with timestamps & signatures).  
- Implement drift remediation with GitOps pattern: auto‑PR to IaC repo to fix drift, manual review required to merge.  
- Use ML to cluster recurring violations and propose higher‑level remediation patterns.

---

### 17. Appendix: Useful Commands & Snippets

- **Run Checkov locally:** `checkov -d ./terraform --quiet --output json > checkov_report.json`  
- **Run InSpec profile:** `inspec exec ./profiles/aws-cis -t aws://`  
- **Test Rego with Conftest:** `conftest test plan.json --policy ./rego`  
- **Run OpenSCAP check:** `oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_standard --results results.xml scap.xml`

---

