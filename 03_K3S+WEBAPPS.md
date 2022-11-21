# Развёртывание приложений в K3s

### Шаг 1. Меняем Vagrantfile

Переместим уже созданную нами конфигурацию в папку p1, так как на текущий момент мы выполнили условия первого пункта сабжа. Копируем эту папку и назовём её p2. Теперь наша задача - модернизировать текущий проект в соответствии со втрорым пунктом задания.

По заданию нам нужен один образ виртуалки с тремя запущенными приложениями. Изменим наш Vagrantfile в соответствии с этой задачей.

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# master config
MASTER_NODE_NAME = 'Kferterb'
MASTER_NODE_HOSTNAME = 'Server'
MASTER_NODE_IP = '192.168.56.110'

# machines config
MEM = 1024
CPU = 1

# create machines config
Vagrant.configure("2") do |config|
	config.vm.box = "bento/debian-11"
	config.vm.provider "virtualbox" do |v|
		v.memory = MEM
		v.cpus = CPU
		# for connect with SSH on both machines with no password
		id_rsa_pub = File.read("#{Dir.home}/.ssh/id_rsa.pub")
  		config.vm.provision "copy ssh public key", type: "shell",
    	  inline: "echo \"#{id_rsa_pub}\" >> /home/vagrant/.ssh/authorized_keys"
	end

  # master node config
	config.vm.define MASTER_NODE_NAME do |master|
		master.vm.hostname = MASTER_NODE_HOSTNAME
		master.vm.network :private_network, ip: MASTER_NODE_IP
		# configure shared folder
		master.vm.synced_folder ".", "/mnt", type: "virtualbox"
		# run script for master node with argument
		master.vm.provision "shell", privileged: true, path: "scripts/master_node_setup.sh", args: [MASTER_NODE_IP]
		master.vm.provider "virtualbox" do |v|
			v.name = MASTER_NODE_NAME
		end
	end
end
```

Теперь наша задача - написать для этой виртуальной машины небольшие пайплайны для запуска приложений. Но для начала изменим скрипт ``master_node_setup.sh``, который будет запускать эти самые пайплайны.

### Шаг 2. Меняем скрипт запуска

Скрипт worker-ноды в p2 можно просто удалить, а вот скрипт мастер-ноды мы изменим:

``nano scripts/master_node_setup.sh``

```
#!/bin/bash

# add k3s in env
export INSTALL_K3S_EXEC="--write-kubeconfig-mode=644 --tls-san $(hostname) --node-ip $1  --bind-address=$1 --advertise-address=$1 "

# download and run k3s agent
curl -sfL https://get.k3s.io |  sh -

# Deployment app1
/usr/local/bin/kubectl apply -f /mnt/confs/app1.yaml -n kube-system

# Deployment app2
/usr/local/bin/kubectl apply -f /mnt/confs/app2.yaml -n kube-system

# Deployment app3
/usr/local/bin/kubectl apply -f /mnt/confs/app3.yaml -n kube-system

#Create Ingress
/usr/local/bin/kubectl apply -f /mnt/confs/ingress.yaml -n kube-system
```

Как видим, мы создали папку с конфигурациями и назвали её confs. В этой директории мы создали три приложения: app1, app2, app3 с соответствующими .yaml - файлами. Ниже я привожу их листинг.

### Шаг 3. Конфигурация app1

``app1.yaml``:

```
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

Точно так же создаём файлы остальных приложений, меняя в нужных местах app1 на app2 и app3. Одному из них, в моём случае второму, задаём три реплики, остальным двум - по одной.

``app2.yaml``:

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: app2
  labels:
    app: app2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: paulbouwer/hello-kubernetes:1.10
        ports:
          - containerPort: 8080
        env:
          - name: MESSAGE
            value: "Hello from app2."

kind: Service
apiVersion: v1
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

``app3.yaml``:

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: app3
  labels:
    app: app3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3
  template:
    metadata:
      labels:
        app: app3
    spec:
      containers:
      - name: app3
        image: paulbouwer/hello-kubernetes:1.10
        ports:
          - containerPort: 8080
        env:
          - name: MESSAGE
            value: "Hello from app3."

kind: Service
apiVersion: v1
metadata:
  name: app3-service
spec:
  selector:
    app: app3
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

### Шаг 4. Конфигурация ingress

Здесь мы прописываем правила для этих трёх хостов.

``ingress.yaml``:

```
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: app-ingress
spec:
  rules:
  - host: app1.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3-service
            port:
              number: 80
```

Наша конфигурация готова. Пробуем её запустить.

### Шаг 5. Запуск

Сначала удалим старые конфигурации:

``vagrant destroy``

Затем запустим новую конфигурацию:

``vagrant up --provider=virtualbox``

У нас должна запуститься наша виртуальная машина с тремя кластерами:


Подключимся и увидим, что всё работает:

