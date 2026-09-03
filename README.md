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

Terraform workflow'u çalıştığında `TF_BACKEND_CONFIG` secret'ındaki backend HCL'i kullanarak init eder; bu taslak oluşturma sırasında hiçbir `terraform init` çalıştırılmamıştır. Apply yalnızca caller açıkça `apply: true` gönderdiğinde ve GitHub Environment koruması geçildiğinde yapılır.
