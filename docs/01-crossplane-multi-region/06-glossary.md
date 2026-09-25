# 06 – Glossary

Bảng thuật ngữ dùng trong series và trong các ghi chú này. Thuật ngữ giữ tiếng Anh vì đó là tên trong tài liệu và trong Kubernetes API. Phần Crossplane theo v2.4 (release mới nhất v2.4.2, 22/9/2026).

## Crossplane

| Thuật ngữ | Giải thích |
|-----------|------------|
| Crossplane | Control plane framework chạy trên Kubernetes. Biến cloud resource và Kubernetes resource thành API có reconcile liên tục, cho platform team xây API hạ tầng mức cao. Trong series chạy ở management cluster. |
| Provider | Package cài vào cluster, mang controller và CRD cho một hệ bên ngoài. Với AWS là `provider-upjet-aws` (v2.8.1). |
| ProviderConfig | Cấu hình xác thực của Provider với cloud, **namespaced** trong v2. |
| ClusterProviderConfig | Bản dùng chung toàn cluster của ProviderConfig. MR không khai báo `providerConfigRef` sẽ dùng `ClusterProviderConfig` tên `default`. |
| Managed Resource (MR) | Một object Kubernetes ứng với đúng một object bên ngoài (một VPC, một EKS cluster). Provider reconcile nó. v2: namespaced. MR cluster-scoped là legacy, sẽ bị gỡ. |
| Managed Resource Definition (MRD) | Cơ chế v2 cho phép chỉ kích hoạt những MR mình cần từ một Provider, giảm tải cluster. |
| CompositeResourceDefinition (XRD) | Định nghĩa một API mới của platform: group, kind, version, field trong spec, và `scope`. Là "interface". |
| scope (của XRD) | `Namespaced` (mặc định, khuyến nghị), `Cluster` (cho resource mức platform như RBAC), `LegacyCluster` (v1, có Claim). |
| Composition | Hiện thực một XRD. v2: luôn là `mode: Pipeline`, một chuỗi Function. Là "implementation". Một XRD có thể có nhiều Composition. |
| Composition Function (Function) | Extension chạy như pod, nhận XR và trả về resource cần tạo. Ví dụ `function-patch-and-transform`, `function-kcl`, `function-go-templating`, `function-python`, `function-auto-ready`. |
| Composite Resource (XR) | Instance của XRD, thứ người dùng tạo. v2: namespaced, có thể compose MR, Kubernetes resource thường và CRD bên thứ ba. `RegionalCluster/prod-eu-west-1` trong bài là một XR. |
| spec.crossplane | Nơi v2 gom toàn bộ máy móc của Crossplane trong XR (compositionRef, resourceRefs...), tách khỏi field nghiệp vụ. |
| Claim | Khái niệm v1. Object namespaced ủy quyền cho một XR cluster-scoped. Chỉ còn khi XRD dùng `scope: LegacyCluster`. Bài blog dùng chữ "claim" theo nghĩa này. |
| Operation | Alpha trong v2. Chạy function pipeline một lần tới khi xong như Job. Có `Operation`, `CronOperation`, `WatchOperation`. |
| DeploymentRuntimeConfig | Cấu hình cách Crossplane chạy pod của Provider và Function. Thay cho `ControllerConfig` đã bị gỡ. |
| Reconcile | Vòng lặp controller so trạng thái thực với trạng thái mong muốn và sửa cho khớp, liên tục. |
| Drift | Trạng thái thực lệch khỏi khai báo, thường do sửa tay. Reconcile liên tục là cách chống drift. |

## Cluster và hạ tầng

| Thuật ngữ | Giải thích |
|-----------|------------|
| Management cluster | Cluster chạy platform control plane: Crossplane, FluxCD, orchestration app, promotion engine. Không chạy application workload. |
| Workload cluster | EKS cluster chạy application, mỗi region một cái: `prod-eu-west-1`, `prod-eu-central-1`, `prod-us-east-1`. Mỗi cái chạy FluxCD riêng. |
| RegionalCluster | Tên API platform trong bài. Một XR đại diện cho toàn bộ stack hạ tầng của một region. |
| Failure boundary | Ranh giới mà lỗi không lan qua. Trong bài, mỗi region là một failure boundary vì XR và Flux độc lập. |
| Active-active | Nhiều region cùng nhận traffic. Failover tự động khi health check fail. |
| Active-passive | Một region nhận toàn bộ traffic, region còn lại chờ. Failover tự động hoặc tay. |
| Weighted routing | Route 53 chia traffic theo trọng số giữa các endpoint. |
| Latency-based routing | Route 53 đưa người dùng tới endpoint có độ trễ thấp nhất. |
| Global Accelerator | Dịch vụ AWS cho anycast IP cố định, định tuyến tới regional endpoint gần nhất còn khoẻ. |
| OIDC provider | Trong ngữ cảnh EKS: cho phép Kubernetes ServiceAccount nhận IAM role (IRSA). Một trong các resource Composition tạo. |

## GitOps và delivery

| Thuật ngữ | Giải thích |
|-----------|------------|
| GitOps | Git là source of truth. Controller trong cluster kéo desired state từ Git và reconcile, không ai apply tay. |
| FluxCD | Bộ controller GitOps cho Kubernetes. Trong series chạy ở cả management cluster và mọi workload cluster. |
| GitRepository | Flux resource khai báo một Git repo là nguồn. |
| Kustomization | Flux resource khai báo path trong một nguồn cần apply vào cluster. |
| HelmRelease | Flux resource khai báo một Helm chart cần cài và giữ đúng values. |
| platform-infra | Repo chứa XR hạ tầng, chia theo `regions/<region>/{cluster,network,dns}.yaml`. Flux ở management cluster reconcile repo này. |
| Fleet repo | Repo chứa config GitOps cho các workload cluster: namespace, RBAC, Flux resource, version app. Orchestration app và promotion engine ghi vào đây. |
| Platform orchestration app | App nội bộ nhận khai báo ngắn từ app team, sinh config Flux mức thấp và commit vào fleet repo. Không apply trực tiếp. |
| Promotion engine | Thành phần quyết định version được đi tới environment hoặc region tiếp theo. Đọc GitHub Deployments, tạo PR vào fleet repo. Không chạy `kubectl apply`. |
| GitHub Deployment | Object trên GitHub biểu diễn yêu cầu deploy một ref tới một environment. Có status `queued`, `in_progress`, `success`, `failure`, `inactive`. |
| Promotion | Đưa cùng một version từ target này sang target tiếp theo (staging → prod-eu-west-1 → prod-eu-central-1). |

## Nguồn

- [Building a Multi-Region EKS Platform with Crossplane, FluxCD, and GitOps](https://medium.com/@kostasdihalas/building-a-multi-region-eks-platform-with-crossplane-fluxcd-and-gitops-c0b3ea4dd567)
- [Provisioning Multi-Region AWS Infrastructure with Crossplane](https://medium.com/@kostasdihalas/provisioning-multi-region-aws-infrastructure-with-crossplane-c5010fb39b5f)
- [Crossplane docs v2.4](https://docs.crossplane.io/latest/) – định nghĩa Provider, ProviderConfig, MR, XRD, Composition, Function, XR, Operation. Không nằm trong bài.
