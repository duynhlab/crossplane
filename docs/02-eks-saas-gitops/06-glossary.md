# 06 – Glossary series EKS SaaS GitOps

Thuật ngữ SaaS và tool xuất hiện trong workshop AWS. Thuật ngữ Crossplane và GitOps chung nằm ở [glossary series 1](../01-crossplane-multi-region/06-glossary.md).

## SaaS và multi-tenancy

| Thuật ngữ | Giải thích |
|-----------|------------|
| Tenant | Một khách hàng (tổ chức) dùng SaaS, có định danh riêng, được onboard và vận hành bằng cùng một cơ chế với tenant khác. |
| Tier | Phân khúc khách hàng theo giá và trải nghiệm (Basic, Advanced, Premium). Khái niệm kinh doanh, ánh xạ sang deployment model. |
| Silo | Tenant có tài nguyên riêng (namespace, deployment, queue, database). Vẫn dùng chung identity và vận hành. |
| Pool | Tenant dùng chung tài nguyên, phân biệt bằng tenant context trong request và dữ liệu. |
| Bridge (hybrid) | Trộn: một số microservice silo, một số pool, quyết định theo từng service. |
| Pool environment | Bộ tài nguyên dùng chung cho tenant Pool, trong workshop là namespace `pool-1` với HelmRelease riêng và Terraform CR riêng. |
| Sharding | Chia tenant Pool ra nhiều pool env (`pool-1`, `pool-2`) để giảm blast radius. |
| Noisy neighbor | Một tenant chiếm tài nguyên làm tenant khác trong cùng pool bị chậm. Lý do chính để đưa một service sang silo. |
| Blast radius | Phạm vi ảnh hưởng khi một thành phần lỗi. Pool có blast radius lớn hơn silo. |
| Onboarding | Tạo tenant mới: tạo file HelmRelease từ template, commit, để Flux và Tofu Controller tạo tài nguyên. |
| Offboarding | Xoá tenant: xoá file HelmRelease và dòng kustomization, Flux prune, Tofu Controller destroy. |
| Control plane (SaaS) | Phần nền tảng lo onboarding, billing, identity, deployment. Trong workshop là thư mục `control-plane/` với Argo và onboarding-service. Khác nghĩa với Kubernetes control plane. |
| Application plane (SaaS) | Phần chạy workload của tenant. Trong workshop là `application-plane/` với tier-templates, pooled-envs, tenants. |
| Staggered deployment (wave) | Rollout theo từng nhóm, ở đây là từng tier: Basic → Advanced → Premium. |
| tenantID header | Header HTTP mà ALB dùng để route request về đúng namespace của tenant. |

## Tofu Controller và Terraform

| Thuật ngữ | Giải thích |
|-----------|------------|
| Tofu Controller | Controller của Flux (dự án flux-iac, trước tên tf-controller) chạy Terraform hoặc OpenTofu từ custom resource `Terraform`. Mới nhất v0.16.5. |
| Terraform CR | Resource `infra.contrib.fluxcd.io/v1alpha2, kind: Terraform` khai báo module, source, vars, chính sách approve. |
| tf-runner | Pod do Tofu Controller tạo cho mỗi Terraform CR để chạy `init`, `plan`, `apply`. |
| approvePlan | Field trong Terraform CR. `auto` là apply ngay; giá trị `plan-main-xxx` là duyệt tay từng plan qua Git. |
| destroyResourcesOnDeletion | Field trong Terraform CR. `true` thì xoá CR là `terraform destroy`. |
| tfstate Secret | State Terraform lưu trong Secret `tfstate-default-<name>` trong `flux-system`. |
| writeOutputsToSecret | Ghi output của module vào Secret để app đọc (queue URL, table name). |
| Drift detection | Tofu Controller plan lại theo `interval`, thấy lệch thì apply sửa. Có thể tắt hoặc chạy chế độ chỉ phát hiện. |
| Branch Planner | Tính năng tech preview của Tofu Controller: chạy plan cho PR và comment kết quả. |
| Terraform module `tenant-apps` | Module hạ tầng per-tenant trong workshop: SQS, DynamoDB, IRSA, sau Lab 5 thêm S3, SSM, IAM policy. Bật tắt bằng `enable_producer`, `enable_consumer`, `enable_payments`. |
| Git tag làm version module | Module Terraform được đóng gói bằng tag `v0.0.1`, `v1.0.0`; mỗi tag một `GitRepository` của Flux. |
| IRSA | IAM Roles for Service Accounts. Cách pod trong EKS nhận IAM role qua OIDC, dùng cho Argo, Tofu Controller, app pod. |

## Argo Events và Argo Workflows

| Thuật ngữ | Giải thích |
|-----------|------------|
| Argo Events | Hệ event-driven cho Kubernetes: nhận event ngoài, chuyển qua bus, trigger hành động. Mới nhất v1.9.11. |
| EventBus | Bus nội bộ của Argo Events (NATS, Jetstream, Kafka). Workshop dùng NATS native 3 replica. |
| EventSource | Kết nối tới nguồn event (SQS, SNS, webhook, cron...) và publish vào EventBus. |
| Sensor | Khai báo dependency (event chờ) và trigger (hành động). Có parameter mapping từ event body sang resource được tạo. |
| Trigger | Hành động của Sensor: tạo Kubernetes resource, gọi HTTP, chạy Argo Workflow... |
| Argo Workflows | Workflow engine chạy từng step là container trên Kubernetes. Mới nhất v4.1.4. |
| WorkflowTemplate | Định nghĩa step tái dùng trong một namespace. Workflow gọi qua `templateRef`. |
| Workflow | Một lần chạy cụ thể, thường `generateName` để tạo tên duy nhất. |
| Mutex, semaphore | Cơ chế `synchronization` giới hạn số Workflow chạy đồng thời. Workshop dùng mutex `workflow` để không có hai workflow cùng push Git. |
| workflow-scripts | Thư mục và image chứa 5 shell script (validate, clone, onboarding, deployment, offboarding) chạy trong Argo. |

## Flux image automation và Helm

| Thuật ngữ | Giải thích |
|-----------|------------|
| ImageRepository | Flux resource quét registry để lấy danh sách tag của một image. |
| ImagePolicy | Flux resource chọn một tag theo quy tắc (semver, alphabetical, numerical) sau khi lọc bằng regex. |
| ImageUpdateAutomation | Flux resource sửa file YAML trong Git tại marker `$imagepolicy`, commit và push. |
| Marker `$imagepolicy` | Comment `# {"$imagepolicy": "ns:policy:tag"}` cạnh field cần cập nhật. |
| `prd-TIMESTAMP` | Quy ước tag image của CI trong workshop, cho phép sắp xếp alphabetical để lấy tag mới nhất. |
| Semver wildcard `0.0.x` | Chart version trong HelmRelease, nhận mọi patch nhưng không nhận minor hay major mới. |
| HelmRepository OCI | Flux source trỏ tới registry OCI (ECR) chứa Helm chart. |
| helm-tenant-chart | Chart duy nhất cho mọi tenant và pool env, có template `terraform.yaml` sinh Terraform CR. |
| application-chart | Chart 1:1 với service không thuộc tenant (onboarding-service). |
| postBuild.substituteFrom | Flux Kustomization thay biến `${...}` bằng giá trị từ ConfigMap hoặc Secret trước khi apply. Workshop dùng ConfigMap `saas-infra-outputs`. |
| saas-infra-outputs | ConfigMap do Terraform bootstrap ghi: account id, ECR URL, Gitea URL, IRSA ARN, queue URL. Cầu nối giữa hạ tầng nền và GitOps. |

## Add-on và công cụ khác trong workshop

| Thuật ngữ | Giải thích |
|-----------|------------|
| Karpenter | Node autoscaler cho Kubernetes trên AWS, ưu tiên Spot. Workshop dùng v0.34.1, mới nhất v1.14.1. |
| AWS Load Balancer Controller | Tạo ALB, NLB từ Ingress và Service. |
| Kubecost | Đo cost theo namespace, hữu ích để tính cost per tenant. |
| Capacitor | Web UI cho Flux. |
| Gitea | Git server tự host chạy trên EC2 trong workshop, kèm Gitea Actions làm CI. |
| Amazon ECR | Registry chứa cả container image và Helm chart OCI. |

## Nguồn

- Workshop [Building SaaS applications on Amazon EKS using GitOps](https://catalog.workshops.aws/eks-saas-gitops/en-US/01-introduction) và repo [aws-samples/eks-saas-gitops](https://github.com/aws-samples/eks-saas-gitops).
- [AWS SaaS Lens](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/silo-pool-and-bridge-models.html), [Tofu Controller docs](https://flux-iac.github.io/tofu-controller/), [Argo Events docs](https://argoproj.github.io/argo-events/), [Argo Workflows docs](https://argo-workflows.readthedocs.io/), [Flux docs](https://fluxcd.io/flux/).
