# Домашнее задание к занятию «Микросервисы: масштабирование» - Юрочкин В.А.

## Задача 1: Кластеризация

### Предложенное решение

**Kubernetes (K8s)** в связке с дополнительными компонентами экосистемы:

| Компонент | Роль в системе |
|---|---|
| **Kubernetes** | Оркестрация контейнеров, управление жизненным циклом подов |
| **Helm** | Пакетный менеджер — управление релизами и конфигурациями приложений |
| **NGINX Ingress Controller** | Маршрутизация входящего трафика, разделение внешнего/внутреннего периметра |
| **Horizontal Pod Autoscaler (HPA)** | Автоматическое горизонтальное масштабирование по метрикам |
| **CoreDNS** | Service Discovery внутри кластера (входит в состав K8s) |
| **Kubernetes Secrets + Sealed Secrets / Vault** | Безопасное хранение чувствительных данных |

---

### Архитектурная схема взаимодействия

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │  Ingress /  │   ← Внешний периметр (NGINX Ingress)
                    │  Load Bal.  │     NodePort / LoadBalancer Service
                    └──────┬──────┘
                           │
          ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
          Kubernetes Cluster│
          ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                    ┌──────▼──────┐
                    │  ClusterIP  │   ← Внутренний периметр
                    │  Services   │     (недоступны снаружи)
                    └──┬──┬──┬───┘
                       │  │  │
              ┌────────┘  │  └────────┐
         ┌────▼───┐  ┌────▼───┐  ┌───▼────┐
         │ Pod A  │  │ Pod B  │  │ Pod C  │
         │(x3 реп)│  │(x2 реп)│  │(x1 реп)│
         └────────┘  └────────┘  └────────┘
              │            │           │
         ┌────▼────────────▼───────────▼────┐
         │         CoreDNS (Service         │
         │         Discovery)               │
         └──────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │  Secrets /  │
                    │  ConfigMaps │
                    └─────────────┘
```

---

### Соответствие требованиям

#### Поддержка контейнеров

Kubernetes — нативный оркестратор контейнеров. Поддерживает образы **Docker / OCI**, управляет жизненным циклом контейнеров (запуск, перезапуск при сбое, обновление без даунтайма через Rolling Update / Blue-Green / Canary стратегии).

```yaml
# Пример Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        - name: my-service
          image: my-registry/my-service:1.2.3
          ports:
            - containerPort: 8080
```

---

#### Обнаружение сервисов и маршрутизация запросов

**CoreDNS** (встроен в K8s) автоматически регистрирует DNS-имена для каждого Service. Любой под может обращаться к другому сервису по имени:

```
http://my-service.my-namespace.svc.cluster.local:8080
```

Для внешней маршрутизации используется **NGINX Ingress Controller** с правилами по path / host:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

---

#### Горизонтальное масштабирование

Масштабирование выполняется командой или через манифест:

```bash
# Ручное масштабирование
kubectl scale deployment my-service --replicas=10
```

```yaml
# Через манифест
spec:
  replicas: 10
```

Kubernetes равномерно распределяет поды по нодам кластера. При добавлении новых нод (cluster autoscaler) поды автоматически размещаются на них.

---

#### Автоматическое масштабирование

**Horizontal Pod Autoscaler (HPA)** масштабирует количество реплик на основе метрик (CPU, Memory, custom metrics через Prometheus Adapter):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

Для масштабирования самих нод кластера используется **Cluster Autoscaler** (интегрируется с AWS EKS, GKE, AKS и bare-metal через Ansible/Terraform).

---

#### Разделение ресурсов: внешние vs внутренние

Kubernetes предоставляет три типа Service для явного разграничения доступности:

| Тип Service | Доступность | Применение |
|---|---|---|
| `ClusterIP` (default) | Только внутри кластера | Межсервисное взаимодействие |
| `NodePort` | Снаружи через порт ноды | Dev/staging среды |
| `LoadBalancer` | Снаружи через L4 балансировщик | Прод при работе с cloud-провайдером |

Дополнительно разграничение обеспечивается через:
- **Namespaces** — логическая изоляция команд/окружений
- **NetworkPolicy** — L3/L4 firewall-правила между подами
- **Ingress** — единственная точка входа снаружи для HTTP/HTTPS трафика

```yaml
# NetworkPolicy: запрет всего входящего трафика, кроме из своего namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: production
```

---

#### Конфигурирование через переменные среды и безопасное хранение секретов

**ConfigMap** — для несекретных конфигурационных данных:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "postgres.production.svc.cluster.local"
```

**Secret** — для чувствительных данных (пароли, ключи, токены):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # base64
  API_KEY: c2VjcmV0a2V5MTIz         # base64
```

Монтирование в под как переменные среды:

```yaml
spec:
  containers:
    - name: my-service
      image: my-registry/my-service:1.2.3
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
```

**Для продакшн-окружений** рекомендуется усиленная защита секретов через один из следующих инструментов:

| Инструмент | Описание |
|---|---|
| **HashiCorp Vault** | Централизованное хранилище секретов с аудитом, динамическими кредами, rotation |
| **Sealed Secrets (Bitnami)** | Шифрует Secret-манифесты для безопасного хранения в Git (GitOps-совместимо) |
| **AWS Secrets Manager / GCP Secret Manager** | Нативные решения для cloud-окружений |
| **External Secrets Operator** | Синхронизирует секреты из внешних хранилищ в K8s Secrets |

---

### Обоснование выбора

**Kubernetes** выбран как де-факто стандарт оркестрации в индустрии по следующим причинам:

1. **Зрелость и экосистема.** K8s существует с 2014 года, поддерживается CNCF, имеет огромное сообщество и тысячи production-готовых интеграций (мониторинг, логирование, service mesh, CI/CD).

2. **Полнота покрытия требований.** Все шесть требований задания закрываются нативными примитивами K8s или стандартными компонентами его экосистемы — без необходимости в экзотических решениях.

3. **Vendor-agnostic.** Решение одинаково работает на любом облаке (EKS, GKE, AKS) и на bare-metal (kubeadm, k3s, RKE2), что исключает vendor lock-in.

4. **Декларативный подход.** Вся инфраструктура описывается YAML-манифестами, что обеспечивает IaC, GitOps (ArgoCD / Flux) и воспроизводимость окружений.

5. **Альтернативы и почему они не выбраны:**

| Альтернатива | Причина отказа |
|---|---|
| **Docker Swarm** | Значительно меньше возможностей, фактически заморожен в развитии |
| **Nomad (HashiCorp)** | Хорошее решение, но меньшая экосистема; нет нативного Service Discovery без Consul |
| **Mesos/Marathon** | Устаревшее решение, потеряло поддержку сообщества |

---

### Итоговый стек

```
Kubernetes (оркестрация)
├── Helm (управление релизами)
├── NGINX Ingress Controller (внешняя маршрутизация)
├── CoreDNS (service discovery, встроен)
├── HPA + Cluster Autoscaler (авто-масштабирование)
├── NetworkPolicy (разделение периметров)
└── Secrets + HashiCorp Vault / Sealed Secrets (управление секретами)
```
