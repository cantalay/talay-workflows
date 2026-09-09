# talay-workflows

Uygulama ve altyapı repolarının çağıracağı reusable GitHub Actions iş akışlarıdır.

- `terraform.yaml`: fmt/validate/plan ve onaylı apply
- `helm.yaml`: dependency, lint, render ve package
- `java.yaml`: Java 25 (LTS), Maven/Gradle test, container build ve Trivy
- `node.yaml`: Node 24, npm/pnpm/yarn test, container build ve Trivy
- `web.yaml`: React/Vite/Expo Web build ve static container
- `expo.yaml`: React Native Android/iOS EAS build
- `promote-image.yaml`: immutable image tag'ını environments reposuna PR olarak taşır

Workflow referanslarını `@main` yerine immutable release tag veya commit SHA ile kullanın. Buradaki action sürümleri 2026-09-03 tarihinde resmi repolarındaki güncel release'lere sabitlenmiştir.

Terraform workflow'u `TF_BACKEND_CONFIG` verilirse bu backend HCL'iyle init eder. Kubernetes backend kullanan repolarda kubeconfig `KUBE_CONFIG` secret'ından geçici dosyaya yazılır ve `KUBE_CONFIG_PATH` ile backend'e sunulur. Bootstrap'ın local backend'i için iki secret da isteğe bağlıdır. Apply yalnızca caller açıkça `apply: true` gönderdiğinde ve GitHub Environment koruması geçildiğinde yapılır.

Java, Node ve Web workflow'larında Trivy HIGH/CRITICAL bulguları build'i durdurur. Taramadan
geçen image publish edilirken BuildKit SBOM ve provenance attestations da GHCR manifestine eklenir.
SARIF yükleme, private repolarda GitHub Code Security lisansı zorunluluğu oluşturmaması için
varsayılan olarak kapalıdır; Code Scanning açık repolar `upload-sarif: true` gönderebilir.
Tüm build ve IaC workflow'ları bağımlılık kurulumundan ve secret dosyaları oluşturulmadan önce
tracked çalışma ağacını Trivy secret scanner ile tarar; bir bulgu pipeline'ı durdurur.

pnpm kullanan Node/Web/Expo çağrıları `pnpm-version` ile sürümü sabitler; pnpm, Node dependency
cache hazırlanmasından önce kurulur. Terraform caller'ları Keycloak ve Vault provider bilgilerini
yalnız reusable workflow secret girişleriyle aktarabilir; bu değerler dosyaya yazılmaz.

Eski veya merkezi private GHCR paketleri repo `GITHUB_TOKEN` erişimi vermiyorsa çağıran repo mevcut `GHCR_PAT` secret'ını opsiyonel `GHCR_TOKEN` olarak geçirir. Yeni paketlerde repository Actions access tanımlanıp kısa ömürlü `GITHUB_TOKEN` tercih edilmelidir.
