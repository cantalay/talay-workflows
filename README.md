# talay-workflows

Uygulama ve altyapı repolarının çağıracağı reusable GitHub Actions iş akışlarıdır.

- `terraform.yaml`: fmt/validate/plan ve onaylı apply
- `helm.yaml`: dependency, lint, render ve package
- `java.yaml`: Java 25 (LTS), Maven/Gradle test, container build ve Trivy
- `node.yaml`: Node 24, npm/pnpm/yarn test, container build ve Trivy
- `python.yaml`: Python 3.12 + uv (`uv.lock`), ruff + pytest, container build ve Trivy
- `web.yaml`: React/Vite/Expo Web build ve static container
- `expo.yaml`: React Native Android/iOS EAS build
- `promote-image.yaml`: yayınlanan image'ın tag + digest'ini talay-environments values dosyalarına yazar (varsayılan doğrudan commit, `mode: pull-request` ile PR); Argo CD otomatik yayına alır

Workflow referanslarını `@main` yerine immutable release tag veya commit SHA ile kullanın. Buradaki action sürümleri 2026-09-03 tarihinde resmi repolarındaki güncel release'lere sabitlenmiştir.

Terraform workflow'u `TF_BACKEND_CONFIG` verilirse bu backend HCL'iyle init eder. Kubernetes backend kullanan repolarda kubeconfig `KUBE_CONFIG` secret'ından geçici dosyaya yazılır ve `KUBE_CONFIG_PATH` ile backend'e sunulur. Bootstrap'ın local backend'i için iki secret da isteğe bağlıdır. Apply yalnızca caller açıkça `apply: true` gönderdiğinde ve GitHub Environment koruması geçildiğinde yapılır.

Java, Node, Python ve Web workflow'larında Trivy HIGH/CRITICAL bulguları build'i durdurur. Taramadan
geçen image publish edilirken BuildKit SBOM ve provenance attestations da GHCR manifestine eklenir.
SARIF yükleme, private repolarda GitHub Code Security lisansı zorunluluğu oluşturmaması için
varsayılan olarak kapalıdır; Code Scanning açık repolar `upload-sarif: true` gönderebilir.
Tüm build ve IaC workflow'ları bağımlılık kurulumundan ve secret dosyaları oluşturulmadan önce
tracked çalışma ağacını Trivy secret scanner ile tarar; bir bulgu pipeline'ı durdurur.
Terraform yapılandırmaları ile Helm'in render ettiği Kubernetes kaynakları ayrıca HIGH/CRITICAL
misconfiguration taramasından geçmeden planlanamaz veya paketlenemez.

pnpm kullanan Node/Web/Expo çağrıları `pnpm-version` ile sürümü sabitler; pnpm, Node dependency
cache hazırlanmasından önce kurulur. Terraform caller'ları Keycloak ve Vault provider bilgilerini
yalnız reusable workflow secret girişleriyle aktarabilir; bu değerler dosyaya yazılmaz.

Eski veya merkezi private GHCR paketleri repo `GITHUB_TOKEN` erişimi vermiyorsa çağıran repo mevcut `GHCR_PAT` secret'ını opsiyonel `GHCR_TOKEN` olarak geçirir. Yeni paketlerde repository Actions access tanımlanıp kısa ömürlü `GITHUB_TOKEN` tercih edilmelidir.

## Otomatik yayına alma

`java.yaml`, `node.yaml`, `python.yaml` ve `web.yaml` push edilen image için `image-tag` (`sha-<7>`) ve `image-digest` çıktısı verir.
Uygulama reposu bunları `promote-image.yaml`'a geçirir:

```yaml
jobs:
  build:
    uses: cantalay/talay-workflows/.github/workflows/node.yaml@<sha>
    with: { image-name: ghcr.io/cantalay/<image>, push-image: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }} }
  promote:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: cantalay/talay-workflows/.github/workflows/promote-image.yaml@<sha>
    with:
      values-files: |
        apps/prod/<project>/<component>/values.yaml
      image-tag: ${{ needs.build.outputs.image-tag }}
      image-digest: ${{ needs.build.outputs.image-digest }}
    secrets:
      APP_ID: ${{ secrets.TALAY_PROMOTER_APP_ID }}
      APP_PRIVATE_KEY: ${{ secrets.TALAY_PROMOTER_PRIVATE_KEY }}
```

GitHub App `cantalay-talay-promoter` yalnız `talay-environments` reposuna Contents: write yetkisiyle kuruludur. Aynı image'ı
kullanan bütün component'ler (`values-files`) tek commit'te güncellenir; eşzamanlı promotion'lar rebase ile yeniden denenir.
