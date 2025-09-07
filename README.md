# GitOps Infrastructure Repository

Этот репозиторий содержит конфигурацию для развертывания Kubernetes кластера с использованием GitOps подхода через Flux CD.

## 📋 Содержание

- [Структура репозитория](#структура-репозитория)
- [Установка Flux CD](#установка-flux-cd)
- [Развертывание инфраструктуры](#развертывание-инфраструктуры)
- [Управление приложениями](#управление-приложениями)
- [Мониторинг](#мониторинг)
- [Полезные команды](#полезные-команды)

## 🏗️ Структура репозитория

```
proj-files/
├── apps/                         # Приложения
│   ├── dev/                      # Development окружение
│   │   ├── namespace.yaml        # Namespace для dev
│   │   ├── kustomization.yaml    # Kustomization для dev приложений
│   │   └── go-test-app/          # Go тестовое приложение
│   │       ├── kustomization.yaml
│   │       ├── app/              # Основное приложение
│   │       ├── config/           # Конфигурация (ConfigMap, Secret)
│   │       ├── postgres/         # PostgreSQL StatefulSet
│   │       ├── redis/            # Redis StatefulSet
│   │       └── ingress/          # Ingress конфигурация
│   ├── prod/                     # Production окружение (пустое)
│   │   ├── namespace.yaml
│   │   └── kustomization.yaml
│   ├── kustomization.yaml        # Корневой kustomization для приложений
│   ├── dev-kustomization.yaml    # Flux Kustomization для dev
│   └── prod-kustomization.yaml   # Flux Kustomization для prod
├── infra/                        # Инфраструктура
│   ├── apps/                     # Приложения инфраструктуры
│   │   └── gitlab/               # GitLab
│   ├── monitoring/               # Мониторинг
│   │   └── prometheus/           # Prometheus + Grafana
│   ├── platform/                 # Платформенные компоненты
│   │   ├── ingress-nginx/        # Ingress Controller
│   │   └── metallb/              # Load Balancer
│   └── storage/                  # Хранилище
│       └── localpath/            # Local Path Provisioner
├── kustomization.yaml            # Корневой kustomization для инфраструктуры
├── gitlab-kustomization.yaml     # Flux Kustomization для GitLab
├── gitrepository.yaml            # Git Repository источник
└── README.md                     # Эта документация
```

## 🚀 Установка Flux CD

### 1. Установка Flux CLI

```bash
# macOS
brew install fluxcd/tap/flux

# Linux
curl -s https://fluxcd.io/install.sh | sudo bash

# Windows
winget install Weaveworks.Flux
```

### 2. Установка Flux в кластер

```bash
# Проверка совместимости кластера
flux check --pre

# Установка Flux
flux install

# Проверка установки
kubectl get pods -n flux-system
```

### 3. Настройка Git Repository

```bash
# Создание GitRepository ресурса
flux create source git infrastructure-repo \
  --url=https://github.com/your-username/your-repo.git \
  --branch=main \
  --interval=1m

# Или применить из файла
kubectl apply -f gitrepository.yaml
```

## Развертывание инфраструктуры

### 1. Развертывание базовой инфраструктуры

```bash
# Применение корневого kustomization
kubectl apply -f kustomization.yaml

### 2. Развертывание компонентов по отдельности

```bash
# Ingress Controller
kubectl apply -f infra/platform/ingress-nginx-kustomization.yaml

# MetalLB Load Balancer
kubectl apply -f infra/platform/metallb-kustomization.yaml

# Local Path Storage
kubectl apply -f infra/storage/localpath-kustomization.yaml

# Prometheus Monitoring
kubectl apply -f infra/monitoring/prometheus-kustomization.yaml

# GitLab
kubectl apply -f gitlab-kustomization.yaml
```

### 3. Проверка статуса

```bash
# Проверка всех kustomizations
kubectl get kustomizations -n flux-system

# Детальная информация
kubectl describe kustomization infrastructure -n flux-system
```

## Управление приложениями

### 1. Развертывание приложений в dev

```bash
# Применение dev kustomization
kubectl apply -f apps/dev-kustomization.yaml

# Проверка статуса
kubectl get pods -n dev
```

### 2. Развертывание приложений в prod

```bash
# Применение prod kustomization
kubectl apply -f apps/prod-kustomization.yaml

# Проверка статуса
kubectl get pods -n prod
```

### 3. Добавление нового приложения

1. Создайте директорию для приложения в `apps/dev/` или `apps/prod/`
2. Добавьте kustomization.yaml файл
3. Обновите kustomization.yaml в соответствующем окружении
4. Закоммитьте изменения - Flux автоматически применит их

## Мониторинг

### Prometheus

- **URL**: `http://prometheus.monitoring.svc.cluster.local:9090`
- **Port Forward**: `kubectl port-forward -n monitoring svc/prometheus-prometheus-kube-prometheus-prometheus 9090:9090`

### Grafana

- **URL**: `http://grafana.monitoring.svc.cluster.local:3000`
- **Port Forward**: `kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80`
- **Default Login**: `admin/prom-operator`

### ServiceMonitor

Для мониторинга приложений созадн ServiceMonitor ресурс:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-servicemonitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
      component: app
  namespaceSelector:
    matchNames:
      - dev
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

## Полезные команды

### Flux

```bash
# Проверка статуса
flux get kustomizations

# Принудительная синхронизация
flux reconcile kustomization infrastructure

# Приостановка/возобновление
flux suspend kustomization infrastructure
flux resume kustomization infrastructure

# Просмотр логов
flux logs --follow
```

### Kubernetes

```bash
# Проверка подов
kubectl get pods -A

# Проверка сервисов
kubectl get svc -A

# Проверка ingress
kubectl get ingress -A

# Логи приложения
kubectl logs -n dev deployment/myapp-api -f

# Описание ресурса
kubectl describe pod -n dev myapp-api-xxx
```

### Отладка

```bash
# Проверка событий
kubectl get events -A --sort-by='.lastTimestamp'

# Проверка логов Flux
kubectl logs -n flux-system deployment/kustomize-controller

# Проверка Git Repository
kubectl describe gitrepository infrastructure-repo -n flux-system
```

## 🔐 Безопасность

### Секреты

⚠️ **ВАЖНО**: Для того чтобы создать dev приложение нужно создать секрет!

```bash
# Создание секрета вручную
kubectl create secret generic myapp-secret -n dev \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=password
```

## 📚 Дополнительные ресурсы

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
- [GitLab Kubernetes Integration](https://docs.gitlab.com/ee/user/clusters/)