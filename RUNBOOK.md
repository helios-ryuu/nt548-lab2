# RUNBOOK — NT548 Lab 02 (Sổ Tay Vận Hành 3 Pipeline CI/CD & Phối Hợp Nhóm)

Tài liệu quy trình chuẩn (SOP) giúp **Tech Lead** vận hành an toàn cả 3 câu của Lab 02 (bảo vệ ngân sách 50 USD trên AWS và bảo vệ tuyệt đối cụm Kubernetes sẵn có trên máy Fedora) và giúp **3 thành viên hỗ trợ** chuẩn bị app mẫu, chạy test, chụp minh chứng, đẩy báo cáo.

---

## SOP 0 — Kiểm Tra Quyền Learner Lab & Làm Mới Secrets (Chạy Đầu Phiên 4h)

### 0.1. Kiểm tra năng lực sandbox (chỉ làm 1 lần ngày 21–22/11/2026)
Sau khi `Start Lab` và nạp `~/.aws/credentials` (region `us-east-1`), kiểm tra ngay các dịch vụ Câu 2 có được Learner Lab cấp phép không:
```bash
aws sts get-caller-identity
aws iam get-role --role-name LabRole --query 'Role.Arn' --output text
aws codecommit list-repositories --region us-east-1
aws codebuild list-projects --region us-east-1
aws codepipeline list-pipelines --region us-east-1
```
- Nếu **tất cả trả về thành công** (hoặc danh sách rỗng `[]`): triển khai Câu 2 đúng theo chuẩn CodeCommit → CodeBuild → CodePipeline.
- Nếu **bị `AccessDenied`** ở CodeCommit hoặc CodePipeline: chụp lại màn hình lỗi ngay để đưa vào mục *Khó khăn* trong Báo cáo Word và kích hoạt phương án dự phòng ở [SOP 2.4](#24-phương-án-dự-phòng-nếu-learner-lab-chặn-quyền-codecommit--codepipeline).

### 0.2. Cập nhật GitHub Repository Secrets (mỗi khi bắt đầu phiên 4h mới)
Vào GitHub repo `nt548-lab2` → **Settings → Secrets and variables → Actions** và cập nhật:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`
- Các biến cấu hình ổn định (chỉ nhập 1 lần): `TF_STATE_BUCKET`, `MY_IP_CIDR`, `EC2_KEY_NAME`.

---

## SOP 1 — Câu 1: Terraform + GitHub Actions + Checkov (3 Điểm)

### 1.1. Bootstrap S3 Remote State (Bắt buộc — KHÔNG dùng local state)
```bash
export AWS_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export TF_STATE_BUCKET="nt548-lab2-tfstate-${ACCOUNT_ID}-${AWS_REGION}"

aws s3 mb "s3://${TF_STATE_BUCKET}" --region "${AWS_REGION}"
aws s3api put-bucket-versioning \
  --bucket "${TF_STATE_BUCKET}" \
  --versioning-configuration Status=Enabled
aws s3api put-public-access-block \
  --bucket "${TF_STATE_BUCKET}" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```
> Cấu hình `backend "s3"` trong `terraform/backend.tf` sử dụng `use_lockfile = true` (Terraform `>= 1.10`) để khóa state trực tiếp trên S3 mà không cần tạo DynamoDB table.

### 1.2. Hardening & Quét Checkov Local trước khi Push
```bash
cd /home/helios/main/projects/nt548-lab2/terraform
checkov -d . --config-file .checkov.yml
```
**Nguyên tắc xử lý cảnh báo Checkov (để ghi điểm tối đa):**
1. **Fix thật trong code** các lỗi bảo mật cơ bản: đóng Default SG (`CKV2_AWS_12`), bật IMDSv2 (`http_tokens = "required"`), mã hóa EBS (`encrypted = true`), không mở SSH `0.0.0.0/0` (`CKV_AWS_24`), tắt public IP mặc định ở Private Subnet.
2. **Chỉ skip có giải thích** trong `.checkov.yml` các rule bất khả thi hoặc tốn phí trên Learner Lab (ví dụ: KMS CMK bị giới hạn IAM, VPC Flow Logs phát sinh thêm IAM Role/CloudWatch). Ghi rõ bảng đối chiếu này vào Báo cáo Word.

### 1.3. Chạy GitHub Actions & Hủy Hạ Tầng Ngay Sau Demo
1. Push nhánh tính năng hoặc tạo PR → GitHub Actions chạy `checkov` + `terraform fmt/validate` + `terraform plan`.
2. Merge vào `main` → GitHub Actions chạy `terraform apply -auto-approve`.
3. Kiểm tra tài nguyên trên AWS Console/CLI & chụp ảnh minh chứng (`q1-*`).
4. **Hủy hạ tầng ngay lập tức:** Vào tab **Actions** → chọn workflow **Terraform CI/CD** → bấm **Run workflow** → chọn `action = destroy`.
5. Xác minh lại bằng [SOP 5](#sop-5--checklist-kiểm-toán-aws-cuối-phiên-chống-cháy-50-usd).

---

## SOP 2 — Câu 2: CloudFormation + CodeCommit + CodeBuild + CodePipeline (3 Điểm)

### 2.1. Tạo CodeCommit Repo & Push bằng `git-remote-codecommit`
Do AWS Academy Learner Lab không cho tạo IAM User (không có Git HTTPS credentials tĩnh), bắt buộc dùng `git-remote-codecommit` với STS credentials hiện hành:
```bash
# Cài đặt công cụ (1 lần)
pip install --user git-remote-codecommit cfn-lint taskcat

# Tạo repository trên CodeCommit (us-east-1)
aws codecommit create-repository \
  --repository-name nt548-lab2-cfn \
  --repository-description "NT548 Lab 02 CloudFormation Nested Stacks" \
  --region us-east-1

# Push mã nguồn CloudFormation lên CodeCommit
git remote add codecommit codecommit::us-east-1://nt548-lab2-cfn || \
  git remote set-url codecommit codecommit::us-east-1://nt548-lab2-cfn
git push codecommit main
```

### 2.2. Kiểm thử Local trước khi chạy CodeBuild (`cfn-lint`, `taskcat`, `package`)
```bash
cd /home/helios/main/projects/nt548-lab2/cloudformation

# 1. Kiểm tra cú pháp CloudFormation
cfn-lint main.yaml templates/*.yaml

# 2. Kiểm thử triển khai bằng Taskcat (KHÓA CHẶT 1 REGION: us-east-1 trong .taskcat.yml)
taskcat test run --no-delete-on-failure=false
```
> **CẢNH BÁO NGÂN SÁCH VỚI TASKCAT:** Taskcat tạo **CloudFormation Stack thật** trên AWS (có cả NAT Gateway + EC2) rồi mới tự xóa!
> - Luôn khóa `regions: [us-east-1]` trong `.taskcat.yml`.
> - Nếu lệnh `taskcat` hoặc CodeBuild bị ngắt giữa chừng, **phải kiểm tra AWS CloudFormation Console ngay** để xóa tay các stack tên `tCaT-*` mồ côi!

### 2.3. Quy trình trong `buildspec.yml` & CodePipeline (`LabRole`)
1. Tạo S3 Artifact/Package bucket (`nt548-lab2-cfn-pkg-<account-id>-us-east-1`).
2. `buildspec.yml` trong **AWS CodeBuild** chạy tuần tự 3 bước:
   - `install / pre_build`: cài `cfn-lint` và `taskcat`, chạy `cfn-lint`.
   - `build`: chạy `taskcat test run` (tạo và tự xóa stack test trên `us-east-1`).
   - `post_build`: chạy **`aws cloudformation package`** để đẩy toàn bộ nested templates lên S3 và xuất artifact `packaged-template.yaml`:
     ```bash
     aws cloudformation package \
       --template-file cloudformation/main.yaml \
       --s3-bucket "${CFN_PACKAGE_BUCKET}" \
       --s3-prefix "packages" \
       --output-template-file packaged-template.yaml
     ```
3. Tạo **AWS CodePipeline** gồm 3 Stage (chọn **`LabRole`** cho cả Service Role của CodeBuild, CodePipeline và CloudFormation Deploy Action):
   - **Source:** AWS CodeCommit (`nt548-lab2-cfn`, branch `main`).
   - **Build:** AWS CodeBuild (chạy `buildspec.yml`, xuất `packaged-template.yaml`).
   - **Deploy:** AWS CloudFormation (`Create or update a stack`, template `BuildArtifact::packaged-template.yaml`).
4. Sau khi pipeline xanh và chụp ảnh minh chứng xong → xóa Production Stack (`nt548-lab2-cfn-prod`), kiểm tra 0 stack `tCaT-*` sót lại theo [SOP 5](#sop-5--checklist-kiểm-toán-aws-cuối-phiên-chống-cháy-50-usd).

### 2.4. Phương án dự phòng nếu Learner Lab chặn quyền CodeCommit / CodePipeline
- Nếu **CodeCommit bị chặn** nhưng CodePipeline chạy được: đổi Stage Source của CodePipeline sang **GitHub (CodeStar Connection)** hoặc **Amazon S3** (zip artifact), giữ nguyên CodeBuild (`cfn-lint` + `taskcat` + `package`) và CloudFormation Deploy.
- Nếu **`LabRole` bị chặn `iam:PassRole` hoặc CodeBuild/CodePipeline bị khóa trong sandbox**: chạy đầy đủ `cfn-lint` + `taskcat` + `aws cloudformation package/deploy` qua **GitHub Actions** (mô phỏng y nguyên 3 stage), đồng thời đính kèm ảnh lỗi `AccessDenied` từ AWS Console vào mục *Khó khăn & Đề xuất* của Báo cáo Word.

---

## SOP 3 — Câu 3: Jenkins + SonarQube + Trivy + Podman Compose + `kind` (4 Điểm — Fedora Local)

> **RÀNG BUỘC AN TOÀN TỐI THƯỢNG TRÊN MÁY FEDORA:**
> 1. **SELinux Enforcing:** Mọi bind mount trong `infra-local/podman-compose.yml` **BẮT BUỘC** có hậu tố **`:Z`** (ví dụ `./jenkins_home:/var/jenkins_home:Z`).
> 2. **Cách ly Kubernetes tuyệt đối:** Cụm `kind` của Lab 02 **CHỈ ĐƯỢC** xuất kubeconfig ra **`~/.kube/lab02-config`**. **TUYỆT ĐỐI KHÔNG** chạy `kind create cluster` thiếu cờ `--kubeconfig ~/.kube/lab02-config` và **KHÔNG** chỉnh sửa `~/.kube/config` mặc định của cụm 3-node sẵn có!

### 3.1. Chuẩn bị môi trường Fedora Host
```bash
# 1. Tham số kernel bắt buộc để SonarQube (Elasticsearch) khởi động thành công
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=131072

# 2. Kích hoạt rootless Podman socket (cho phép Jenkins / công cụ build giao tiếp nếu cần)
systemctl --user enable --now podman.socket
ls -l "${XDG_RUNTIME_DIR}/podman/podman.sock"

# 3. Kiểm tra phiên bản công cụ
podman --version
podman-compose --version
kind version
kubectl version --client
trivy --version
```

### 3.2. Khởi tạo cụm `kind` cách ly hoàn toàn (`~/.kube/lab02-config`)
```bash
# Lưu timestamp gốc của ~/.kube/config để đối chiếu sau khi tạo cluster
ls -l ~/.kube/config

# Tạo cụm kind trên Podman và xuất kubeconfig RIÊNG ra ~/.kube/lab02-config
KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster \
  --name nt548-lab2 \
  --config /home/helios/main/projects/nt548-lab2/infra-local/kind-config.yaml \
  --kubeconfig ~/.kube/lab02-config

# Xác nhận ~/.kube/config mặc định KHÔNG hề bị đụng chạm
ls -l ~/.kube/config

# Xác minh cụm kind mới bằng file kubeconfig riêng
kubectl --kubeconfig ~/.kube/lab02-config cluster-info
kubectl --kubeconfig ~/.kube/lab02-config get nodes
```

> **Quy tắc sử dụng `kubectl` cho Lab 02:** Trong terminal làm Lab 02, chỉ đặt biến môi trường cục bộ cho riêng tab đó:
> ```bash
> export KUBECONFIG=~/.kube/lab02-config
> kubectl config current-context   # Phải hiện: kind-nt548-lab2
> ```

### 3.3. Khởi động Jenkins & SonarQube bằng Podman Compose (`:Z`)
```bash
cd /home/helios/main/projects/nt548-lab2/infra-local

# Tạo trước các thư mục dữ liệu local (đã nằm trong .gitignore)
mkdir -p jenkins_home sonarqube_data sonarqube_logs sonarqube_extensions postgres_data

# Khởi chạy stack (mọi volume trong podman-compose.yml phải có :Z)
podman-compose up -d
podman-compose ps
```
- **Jenkins UI:** `http://localhost:8080` (lấy mật khẩu lần đầu bằng `podman exec <jenkins-container> cat /var/jenkins_home/secrets/initialAdminPassword`).
- **SonarQube UI:** `http://localhost:9000` (mặc định `admin` / `admin`, đổi mật khẩu và tạo Project Token).

### 3.4. Cấu hình Credentials trong Jenkins & Chạy Pipeline
Vào **Manage Jenkins → Credentials → System → Global credentials** và tạo:
1. `dockerhub-creds` *(Username with password)*: Docker Hub username + Access Token.
2. `sonarqube-token` *(Secret text)*: Token sinh từ SonarQube UI.
3. `lab02-kubeconfig` *(Secret file)*: Upload bản kubeconfig của cụm `kind` (`~/.kube/lab02-config`, lưu ý chỉnh `server:` sang địa chỉ container network hoặc `host.containers.internal` để container Jenkins gọi được API server của `kind`).

Tạo Jenkins Pipeline trỏ tới `microservices/Jenkinsfile` và chạy kiểm chứng đủ 5 stage:
1. **Checkout:** Pull mã nguồn từ GitHub `nt548-lab2`.
2. **Unit Test & SonarQube:** Chạy unit tests (`pytest` / `npm test`) + `sonar-scanner` + kiểm tra **Quality Gate**.
3. **Build Container Image:** Build OCI images cho các microservices (dùng Buildah / Podman socket / Kaniko tương thích Fedora SELinux).
4. **Security Scan (Trivy):** Quét lỗ hổng `HIGH,CRITICAL` trên từng image vừa build (`trivy image ...`), xuất báo cáo.
5. **Push Docker Hub & Deploy `kind`:** Push tag image lên Docker Hub → `kubectl --kubeconfig "$KUBECONFIG_FILE" apply -f microservices/k8s/` → `kubectl rollout status` → smoke test HTTP endpoint của các microservices.

### 3.5. Dừng & Dọn dẹp môi trường Local sau khi demo xong
```bash
# Dừng Jenkins & SonarQube
cd /home/helios/main/projects/nt548-lab2/infra-local
podman-compose down

# Xóa cụm kind riêng của Lab 02 (chỉ xóa cluster nt548-lab2 và file lab02-config)
KIND_EXPERIMENTAL_PROVIDER=podman kind delete cluster \
  --name nt548-lab2 \
  --kubeconfig ~/.kube/lab02-config
rm -f ~/.kube/lab02-config
```

---

## SOP 4 — Hướng Dẫn Cho 3 Thành Viên Hỗ Trợ (Minh Chứng & Git Tối Giản)

### 4.1. Checklist ảnh minh chứng (`docs/evidence/`)
- **Câu 1 (`docs/evidence/q1/`):** `q1-01-s3-state.png`, `q1-02-gh-secrets.png`, `q1-03-checkov-pass.png`, `q1-04-gha-apply-green.png`, `q1-05-aws-resources.png`, `q1-06-gha-destroy-green.png`
- **Câu 2 (`docs/evidence/q2/`):** `q2-01-codecommit-repo.png`, `q2-02-codebuild-cfnlint-taskcat.png`, `q2-03-codepipeline-green.png`, `q2-04-cfn-nested-deployed.png`, `q2-05-cleanup.png`
- **Câu 3 (`docs/evidence/q3/`):** `q3-01-podman-compose-up.png`, `q3-02-kind-isolated-nodes.png`, `q3-03-jenkins-stage-view.png`, `q3-04-sonarqube-quality-gate.png`, `q3-05-trivy-scan-report.png`, `q3-06-dockerhub-images.png`, `q3-07-k8s-pods-curl-ok.png`

### 4.2. Phân vùng thư mục không xung đột
- **TV1 — Ngô Tiến Sỹ (`23521367`, Tech Lead):** `.github/`, `terraform/`, `cloudformation/`, `infra-local/`, `microservices/Jenkinsfile`, `microservices/k8s/`
- **TV2 — Trần Hồng Phúc (`23521222`, App Dev & Test Support):** `microservices/service-*` (source code app mẫu, unit test, `Dockerfile`) & `docs/evidence/test-logs/`
- **TV3 — Điền Quốc Huy (`23520600`, QA & Evidence):** `docs/evidence/`
- **TV4 — Đặng Thế Long (`22520804`, Diagrams, Report & Docs):** `docs/diagrams/`, `docs/report/`, cập nhật tiến độ `WBS.md`

### 4.3. Cheatsheet Git 5 Lệnh Tối Giản (Cho Phúc, Huy, Long)
```bash
git pull --rebase origin main
git status
git add <thu-muc-cua-ban>/
git commit -m "docs(q3): add jenkins and sonarqube screenshots"
git push origin main
```

---

## SOP 5 — Checklist Kiểm Toán AWS Cuối Phiên (Chống Cháy 50 USD)

Chạy cuối mỗi phiên làm Câu 1 hoặc Câu 2 — đặc biệt bắt buộc sau khi chạy **Taskcat**:
```bash
echo "=== 1. EC2 ==="
aws ec2 describe-instances --filters "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}' --output table

echo "=== 2. NAT Gateways (NGỐN TIỀN NGẦM!) ==="
aws ec2 describe-nat-gateways --filter "Name=state,Values=pending,available" \
  --query 'NatGateways[].{ID:NatGatewayId,State:State}' --output table

echo "=== 3. Elastic IPs mồ côi ==="
aws ec2 describe-addresses --query 'Addresses[].{AllocId:AllocationId,IP:PublicIp,Assoc:AssociationId}' --output table

echo "=== 4. CloudFormation Stacks (gồm cả tCaT-* của Taskcat!) ==="
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE ROLLBACK_COMPLETE CREATE_IN_PROGRESS DELETE_FAILED \
  --query 'StackSummaries[].{Name:StackName,Status:StackStatus}' --output table
```
