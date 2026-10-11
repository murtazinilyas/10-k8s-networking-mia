# Домашнее задание к занятию «Как работает сеть в K8s» Муртазин И.А. FOPS-42 SHKUBER-30

### Цель задания

Настроить сетевую политику доступа к подам.

### Чеклист готовности к домашнему заданию

1. Кластер K8s с установленным сетевым плагином Calico.

-----

### Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.
2. В качестве образа использовать network-multitool.
3. Разместить поды в namespace App.
4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.
5. Продемонстрировать, что трафик разрешён и запрещён.

### Решение 1.

Установил minikube, так как по умолчанию на нем стоит сетевой плагин `Kindnet` запустил его командой
```bash
minikube start --network-plugin=cni --cni=calico -p cluster-calico
```

Далее создал Custom Resource Definitions 
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml
```

После установил оператор Tigera 
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
```

И создал custom resource для Calico 
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml
```

Написал [манифесты](https://github.com/murtazinilyas/07-k8s-helm-mia/blob/main/manifests):

<details>  
<summary> namespace.yaml </summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
```

</details>

<details>  
<summary> deployment.yaml </summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mia-frontend
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mia-frontend
  template:
    metadata:
      labels:
        app: mia-frontend
    spec:
      containers:
      - name: network-multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mia-backend
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mia-backend
  template:
    metadata:
      labels:
        app: mia-backend
    spec:
      containers:
      - name: network-multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mia-cache
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mia-cache
  template:
    metadata:
      labels:
        app: mia-cache
    spec:
      containers:
      - name: network-multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
```

</details>

<details>  
<summary> services.yaml </summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mia-frontend-svc
  namespace: app
spec:
  type: ClusterIP
  selector:
    app: mia-frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mia-backend-svc
  namespace: app
spec:
  type: ClusterIP
  selector:
    app: mia-backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mia-cache-svc
  namespace: app
spec:
  type: ClusterIP
  selector:
    app: mia-cache
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

</details>

<details>  
<summary> network-policies.yaml </summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-ingress-on-frontend
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: mia-frontend
  policyTypes:
  - Ingress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: mia-backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: mia-frontend
    ports:
    - protocol: TCP
      port: 80

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-cache
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: mia-cache
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: mia-backend
    ports:
    - protocol: TCP
      port: 80
```

</details>

Запустил все командой

```bash
kubectl apply -f .
```

Проверил доступы к сервисам по условию задания:

![1](https://github.com/murtazinilyas/07-k8s-helm-mia/blob/main/screenshots/1.png)