# Solution

## Steps 6, 7
Запускаем БД снаружи кластера, на внешнем IP-адресе машины, в моем случае - 192.168.0.83, пишем и поднимаем в докере такой docker-compose-pg-only.yml:
```
version: '3.8'
services:
  db:
    image: postgres:12.0-alpine
    ports:
      - "192.168.0.83:5432:5432"
    environment:
      POSTGRES_DB: ${POSTGRES_DB-test_k8s}
      POSTGRES_USER: ${POSTGRES_USER-test_k8s}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD-OwOtBep9Frut}
    volumes:
      - db_data:/var/lib/postgresql/data
volumes:
  db_data:
```
В .env для билда контейнера Django прописываем соотв. URL для запущенной выше базы:
```
DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@192.168.0.83:5432/test_k8s
```
Направляем PowerShell использовать докер демон внутри миникуба:
```
& minikube -p minikube docker-env --shell powershell | Invoke-Expression
```
И в этом терминале билдим Джангу:
```
docker build -t django_app:01 .
```
Далее можем запустить джангу в поде миникуба:
```
apiVersion: v1
kind: Pod
metadata:
  name: django-pod
spec:
  containers:
    - name: django-container
      image: django_app:01
      imagePullPolicy: Never
      ports:
        - containerPort: 8000
```
Запуск пода:
```
kubectl apply -f django-pod.yaml
```
Добавляем port-forward:
```
kubectl port-forward django-pod 8000:80
```
Проверяем, что можем подключиться к поду и запустить там ./manage.py shell:
```
kubectl exec -it django-pod -- /bin/bash
./manage.py shell
```
Если не забыли поднять контейнер с БД, то на странице http://127.0.0.1:8000/admin/ видим админку. А в джанго шелл можем получить список пользователей:
```
>>> from django.contrib.auth.models import User
>>> User.objects.all()
<QuerySet [<User: admin>]>
```

## Step 8

### Вариант с type: NodePort

**(Пришлось временно(?) перебилдить django_app:01 с указанием ALLOWED_HOSTS = ["*"])**

Удаляем под из предыдущего шага:
```
kubectl delete -f django-pod.yaml
```
Генерим yaml деплоя джанги:
```
kubectl create deployment django-deploy --image=django_app:01 --dry-run=client -o yaml > django-deploy.yaml
```
Генерим yaml nodeport-сервиса для доступа к деплойным подам:
```
kubectl create service nodeport django-service --tcp=80:80 --node-port=30080 --dry-run=client -o yaml > django-service.yaml
```
Нужно указать (исправить) конкретное приложение (app) с использованием меток (labels) и селекторов (selectors). В сгенерированном Service исправим app в selectors на наш django-deploy, тогда сервис будет выбирать только поды с меткой app: django-deploy и открывать для них NodePort, чтобы обеспечить доступ к нашему конкретному приложению.
```
...
  selector:
    app: django-deploy
...
```
Запускаем:
```
kubectl apply -f django-deploy.yaml
kubectl apply -f django-service.yaml
```
Узнаём minikube ip (192.168.59.100) и проверяем (не забыв поднять контейнер с PostgreSQL), что админка работает на http://192.168.59.100:30080/admin/

### Вариант с type: LoadBalancer 
...

### Вариант с type: Ingress 
...

## Step 11
Создаём ConfigMap yaml на основе нашего .env:
```
kubectl create cm django-cm --from-env-file=backend_main_django/.env --dry-run=client -o yaml > django-cm.yaml
```
В django-deploy добавляем envFrom configMapRef
```
        envFrom:
          - configMapRef:
              name: django-cm
``` 
Сбилдим образ без .env файла (для чистоты эксперимента), назовём с тегом 02 django_app:02, запушим новый образ в minikube, исправим django-deploy, чтобы использовал django_app:02.
```
& minikube -p minikube docker-env --shell powershell | Invoke-Expression
docker build -t django_app:02 .
minikube image ls
```

Сначала применяем ConfigMap, потом остальное:
```
kubectl apply -f django-cm.yaml
kubectl apply -f django-deploy.yaml
kubectl apply -f django-service.yaml
```

## Step 12

Создадим Ingress:
```
kubectl create ingress django-ingress --rule="star-burger.test/=django-service:80" --dry-run=client -o yaml > django-ingress.yaml
```

Добавим в hosts: 192.168.59.100 star-burger.test

Проверим name, port, изменим pathType на Prefix:
```
      - backend:
          service:
            name: django-service
            port:
              number: 80
        path: /
        pathType: Prefix
```
Применим ingress, предварительно применив всё предыдущее:
```
kubectl apply -f django-cm.yaml
kubectl apply -f django-deploy.yaml
kubectl apply -f django-service.yaml
kubectl apply -f django-ingress.yaml
```
Зайдём в админку: http://star-burger.test/admin/