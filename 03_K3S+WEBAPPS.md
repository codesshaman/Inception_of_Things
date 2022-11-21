# Развёртывание приложений в K3s

### Шаг 1. Копирование конфигурации

Переместим уже созданную нами конфигурацию в папку p1, так как на текущий момент мы выполнили условия первого пункта сабжа. Копируем эту папку и назовём её p2. Теперь наша задача - модернизировать текущий проект в соответствии со втрорым пунктом задания.

Первым делом в нашей новой папке p2 создадим директорию confs, где будем хранить конфигурации наших приложений. Это будут небольшие .yaml - файлы, содержащие всю информацию о наших образах.

создадим в confs файл app1.yaml со следующим содержимым:

```
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: app1
  labels:
    app: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: paulbouwer/hello-kubernetes:1.10
        ports:
          - containerPort: 8080
        env:
        - name: MESSAGE
          value: "Hello from app1."

--- 
kind: Service
apiVersion: v1
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

Это контейнер а-ля "Hello, World!" для кубера. Попробуем его запустить.

### Шаг 2. Запуск приложения

