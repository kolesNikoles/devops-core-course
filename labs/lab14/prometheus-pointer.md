# Lab 14 — ориентир по Prometheus внутри кластера (только бонус)

В Lab 8 Prometheus поднимался стеком **Docker-Compose** на вашем ноутбуке. Бонусный
`AnalysisTemplate` в Lab 14 работает **внутри** кластера k3d — `http://localhost:9090`
из Pod означает *localhost самого Pod*, а не ноутбука.

Здесь три способа дать Argo Rollouts доступный URL. Выберите один
и зафиксируйте выбор в `docs/LAB14.md`.

---

## Вариант 1 — достучаться до хоста (дешевле всего)

k3d отдаёт Docker-хост как **`host.k3d.internal`**. Prometheus из Lab 8 на
`localhost:9090` из любого Pod кластера доступен как
`http://host.k3d.internal:9090`.

```yaml
  provider:
    prometheus:
      address: http://host.k3d.internal:9090
```

✅ Ноль подготовки — работает сразу, если стек Lab 8 поднят через `docker compose up`.
❌ Lab 16 заменит это, когда вы перенесёте Prometheus в кластер — `AnalysisTemplate`
   придётся править снова.
❌ На голом `kind` не работает (там берите `host.docker.internal`).

---

## Вариант 2 — минимальный Prometheus в кластере (рекомендуется)

Поднимите маленький Prometheus в namespace `monitoring`: он скрейпит
Service `app-python` по шаблону аннотации `prometheus.io/scrape`. Именно это
Lab 16 заменит на kube-prometheus-stack — установка сейчас это репетиция
из двух файлов.

```bash
kubectl create namespace monitoring

# Minimal scrape config — adapt the job_name to match your app's Service labels
cat <<'EOF' | kubectl apply -n monitoring -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: app-python
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
            regex: app-python
            action: keep
          - source_labels: [__meta_kubernetes_pod_ip]
            target_label: __address__
            replacement: ${1}:5000
EOF

kubectl apply -n monitoring -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector: { matchLabels: { app: prometheus } }
  template:
    metadata: { labels: { app: prometheus } }
    spec:
      serviceAccountName: prometheus
      containers:
        - name: prometheus
          image: prom/prometheus:v3.1.0
          args:
            - --config.file=/etc/prometheus/prometheus.yml
          ports: [ { containerPort: 9090 } ]
          volumeMounts:
            - { name: config, mountPath: /etc/prometheus }
      volumes:
        - { name: config, configMap: { name: prometheus-config } }
---
apiVersion: v1
kind: Service
metadata: { name: prometheus }
spec:
  selector: { app: prometheus }
  ports: [ { port: 9090, targetPort: 9090 } ]
EOF

# RBAC for the kubernetes_sd_configs discovery — Prometheus needs to list pods
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata: { name: prometheus, namespace: monitoring }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: { name: prometheus }
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: { name: prometheus }
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - { kind: ServiceAccount, name: prometheus, namespace: monitoring }
EOF
```

Дальше в вашем `AnalysisTemplate`:

```yaml
  provider:
    prometheus:
      address: http://prometheus.monitoring:9090
```

✅ Переживёт Lab 16 — kube-prometheus-stack публикует Service с тем же именем
   (`prometheus-operated.monitoring`). Переключение — одна строка в URL.
✅ Нет привязки к хосту — работает на `kind`, EKS, GKE, где угодно.

---

## Вариант 3 — пропустить бонус

Основным заданиям (1–4) Prometheus не нужен. Если стек Lab 8 уже снесён, а
до Lab 16 ещё две недели, сдайте 10/10 по основным заданиям и вернитесь к
бонусу, когда появится kube-prometheus-stack.

---

## Быстрая проверка, какой бы вариант вы ни выбрали

Из одноразового Pod внутри кластера докажите, что URL резолвится и отдаёт
собственные метрики Prometheus:

```bash
kubectl run probe --rm -it --image=curlimages/curl:8.11.0 --restart=Never -- \
  curl -s http://<your-address>/-/healthy
# expected: Prometheus is Healthy.

kubectl run probe --rm -it --image=curlimages/curl:8.11.0 --restart=Never -- \
  curl -s 'http://<your-address>/api/v1/query?query=up' | head -20
# expected: JSON with "status":"success" — proves the API works from inside the cluster
```

Если второй вызов не вернул ряд `up{job="app-python"}`, Prometheus не
скрейпит ваше приложение — сначала почините scrape. `AnalysisTemplate` не
пройдёт на метриках, которых нет.
