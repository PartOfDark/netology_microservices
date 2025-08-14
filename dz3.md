# Домашнее задание: «Микросервисы — масштабирование» (инфраструктурное предложение)

## Рекомендованное решение: Kubernetes (Managed: EKS/GKE/AKS) + стандартные аддоны

**Состав:**
- **Kubernetes** (управляемый кластер в облаке: EKS/GKE/AKS) — платформа оркестрации контейнеров.
- **Container Registry** (ECR/GCR/ACR или GitLab/GHCR) — хранение образов.
- **Ingress Controller** (NGINX Ingress Controller/Traefik) — входной трафик L7.
- **Service Mesh (опционально)**: Istio/Linkerd — для продвинутого трафик-менеджмента.
- **ExternalDNS** — автоматическое управление DNS-записями.
- **cert-manager** — автоматическая выдача TLS-сертификатов (Let’s Encrypt/ACM).
- **Secrets/Config**: Kubernetes Secrets/ConfigMaps + External Secrets Operator (с **HashiCorp Vault**/AWS Secrets Manager/Google Secret Manager).
- **Autoscaling**: HPA/VPA + Cluster Autoscaler — горизонт/вертикаль подов и масштабирование узлов.
- **Сетевые политики**: NetworkPolicy + CNI (Calico/Cilium) — сегментация и безопасность L3/L4.
- **RBAC/Namespaces/Quotas** — многоарендность и изоляция ресурсов.

---

## Как решение закрывает требования

### 1) Поддержка контейнеров
- Kubernetes нативно управляет контейнерами (Docker/Containerd/CRI-O). CI/CD публикует образы в Registry, деплой — через Deployments/StatefulSets/DaemonSets.

### 2) Обнаружение сервисов и маршрутизация
- **Service Discovery**: встроенный **ClusterDNS (CoreDNS)** — каждый Service получает стабильное DNS‑имя (`svc.namespace.svc.cluster.local`).
- **Внутренняя маршрутизация**: `Service` (ClusterIP) для L4, авто‑балансировка на Pod’ы по селекторам.
- **Внешняя маршрутизация**: `Ingress` + Ingress Controller (HTTP/HTTPS, host/path‑based routing).
- **Продвинутый трафик (опционально)**: Service Mesh (канареечные релизы, A/B, mTLS, retry, timeout, circuit breaking).

### 3) Горизонтальное масштабирование
- **ReplicaSets/Deployments** — ручное увеличение `replicas`.
- **HPA** (Horizontal Pod Autoscaler) — авто‑масштабирование Pod’ов по CPU/Memory и пользовательским метрикам (Prometheus Adapter).
- **Cluster Autoscaler** — авто‑масштабирование количества узлов (добавляет/удаляет VM‑ноды при дефиците/избытке ресурсов).

### 4) Автоматическое масштабирование
- Связка **HPA + Cluster Autoscaler** обеспечивает масштабирование на уровне Pod’ов и кластера.
- **VPA** (Vertical Pod Autoscaler) — рекомендация/авто‑подстройка ресурсов Pod’ов (опционально, аккуратно в проде).

### 5) Явное разделение ресурсов, доступных извне и внутри
- **Внутренние сервисы**: `Service` типа **ClusterIP** — доступ только внутри кластера.
- **Внешний доступ**:
  - `Ingress` + Ingress Controller (рекомендуется для HTTP/HTTPS).
  - `Service` типа **LoadBalancer** (L4) — прямой публичный/внешний балансировщик от облака.
  - `NodePort` (реже, для нестандартных кейсов).
- **Namespaces + NetworkPolicy** — сегментация трафика между сервисами/командами/окружениями.
- **ExternalDNS** — управляет DNS‑именами только для тех `Ingress/Service`, которые помечены аннотациями (строгое разграничение).

### 6) Конфигурация через переменные среды + безопасное хранение секретов
- **ConfigMap** — несекретная конфигурация (env/envFrom/volume).
- **Secret** — чувствительные данные: пароли, токены, ключи.
- **External Secrets Operator (ESO)** — синхронизация Kubernetes Secrets с внешними секрет‑хранилищами:
  - **HashiCorp Vault** (динамические креды, audit, политики),
  - **AWS Secrets Manager / Parameter Store**, **Google Secret Manager**, **Azure Key Vault**.
- **RBAC** + ограниченный доступ к Secret’ам по Namespace/Role/RoleBinding.
- **Подпись/шифрование образов и артефактов** (Cosign/KMS) — дополнительный уровень доверия supply‑chain.

---

## Принципы взаимодействия компонентов (поток запроса и деплоя)

1. **Деплой/обновление**: CI/CD публикует образ → Helm/Kustomize применяют манифесты → Kubernetes планирует Pod’ы на ноды, подключает ConfigMaps/Secrets, проставляет env, health‑checks (liveness/readiness), sidecar’ы (mesh).
2. **Service Discovery**: Pod регистрируется за счёт селекторов; трафик идёт через `Service` → kube-proxy/iptables/ebpf маршрутизирует на доступные Pod’ы.
3. **Внешний доступ**: Клиент → DNS (ExternalDNS управляет записями) → cloud LB → Ingress Controller → Service → Pod. `cert-manager` оформляет/обновляет TLS.
4. **Масштабирование**: Метрики → HPA принимает решение увеличить/уменьшить `replicas`; при нехватке ресурсов Cluster Autoscaler добавляет ноды.
5. **Секреты/конфиги**: ESO периодически/по событию синхронизирует секреты из Vault/Cloud Secret Manager в Kubernetes Secrets; Pod получает их как env/volume.

---

## Дополнительные преимущества и операционные аспекты

- **Надёжность и обновления**: RollingUpdate/Blue‑Green/Canary (через Ingress annotations/Service Mesh/Argo Rollouts).
- **Изоляция и квоты**: `ResourceQuota`, `LimitRange`, `PodSecurity/PSA`, `NetworkPolicy`.
- **Наблюдаемость**: Prometheus/Grafana/Alertmanager (метрики), OpenTelemetry/Jaeger/Tempo (трейсы), Loki/OpenSearch (логи).
- **Безопасность сети**: CNI Calico/Cilium (NetworkPolicy, eBPF‑ускорение, политика на L3/L4/L7 — у Cilium).
- **Стоимость и масштаб**: Managed Kubernetes уменьшает операционные риски и TCO (контроль‑плейн SLA, автопатчи).

---

## Альтернативы (если Kubernetes избыточен/ещё рано)

- **HashiCorp Nomad + Consul + Traefik**:
  - Контейнеры: Nomad.
  - Service Discovery: Consul.
  - Маршрутизация: Traefik.
  - Масштабирование: Nomad scaling + autoscaler.
  - Секреты: Vault (нативная интеграция).  
  Подходит для простого/гетерогенного ворклоада (включая non‑container), но экосистема менее стандартизована, чем у K8s.

- **Docker Swarm**:
  - Проще старт, но ограниченные возможности mesh‑сети, секретов и autoscaling (в сравнении с K8s). Рекомендую только для небольших нагрузок и краткоживущих проектов.

---

## Итог

**Kubernetes (управляемый в облаке) + Ingress Controller + ExternalDNS + cert‑manager + HPA/VPA + Cluster Autoscaler + External Secrets Operator (+ Vault/Cloud Secret Manager)** — системное, промышленное решение, которое:  
- нативно работает с контейнерами;  
- обеспечивает сервис‑дискавери и маршрутизацию;  
- масштабируется горизонтально/автоматически;  
- разделяет внутренние и внешние ресурсы (Service/Ingress/NetworkPolicy);  
- поддерживает конфигурацию через переменные среды и безопасное хранение секретов;  
- остаётся де‑факто стандартом индустрии для микросервисов с сильной экосистемой и поддержкой облаков.
