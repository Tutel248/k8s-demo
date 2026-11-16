# Звіт до Практичної роботи 10: Kubernetes, практична частина

## Рівень 60 балів: Локальний запуск, Docker та базовий деплой у Minikube

### 1. Локальний запуск та перевірка ендпоінтів `k8s-demo`
Після клонування форку репозиторію та встановлення залежностей (`npm install`, `npm run dev`) перевірено роботу всіх маршрутів на `http://localhost:8080`:
* `GET /api/status` — повертає JSON зі статусом роботи сервісу, версією та активним режимом `APP_MODE`.
* `GET /api/items?filter=a` — повертає відфільтрований за параметром `filter` список тестових елементів.
* `GET /api/slow` — симулює повільну обробку запиту із затримкою перед відповіддю.
* `GET /api/error` — симулює внутрішній збій сервера (повертає HTTP `500 Internal Server Error`).
* `GET /healthz` — ендпоінт перевірки живості процесу (Liveness probe, повертає `200 OK`).
* `GET /ready` — ендпоінт перевірки готовності приймати трафік (Readiness probe, повертає `200 OK`).

### 2. Контейнеризація через Docker
```powershell
docker build -t k8s-demo-app:1.0.0 .
docker run --rm -p 8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
```
Приклад відповіді ендпоінту `/api/status` із запущеного Docker-контейнера:
```json
{"status":"ok","service":"k8s-demo-api","mode":"dev"}
```

### 3. Запуск кластера Minikube та розгортання маніфестів
* **Активний контекст `kubectl`:** `minikube`
* **Статус ноди:** `minikube` — `Ready` (роль `control-plane`)
* Для роботи в локальному кластері образ завантажено в кеш Minikube (`minikube image load maxnavrotsky/k8s-demo-app:1.0.0`) і застосовано маніфести `namespace.yaml`, `configmap.yaml`, `secret.yaml`, `deployment.yaml`, `service.yaml`.

Вивід стану розгорнутих ресурсів (`kubectl get deployment,rs,pods,svc -n k8s-demo`):
```text
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-api   2/2     2            2           19s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-api-5c45559886   2         2         2       19s

NAME                                READY   STATUS    RESTARTS   AGE
pod/k8s-demo-api-5c45559886-kx2xh   1/1     Running   0          19s
pod/k8s-demo-api-5c45559886-ngxzv   1/1     Running   0          19s

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-api   ClusterIP   10.107.239.245   <none>        80/TCP    19s
```

### 4. Перевірка доступу через `kubectl port-forward`
```powershell
kubectl port-forward svc/k8s-demo-api -n k8s-demo 8080:80
curl.exe http://localhost:8080/api/status
```
**Шлях запиту:** локальний порт `localhost:8080` тунелюється через Kubernetes API Server до `Service` (`k8s-demo-api`, тип `ClusterIP`), який балансує трафік на порт `8080` одного з активних `Pod`-ів.

---

## Рівень 75 балів: Робота з ConfigMap, Secret та Health-checks

### 1. Вплив зміни `ConfigMap` без перезапуску та після `rollout restart`
1. Перевірка початкового значення `APP_MODE` у працюючому pod-і:
   ```powershell
   kubectl exec -n k8s-demo k8s-demo-api-5c45559886-kx2xh -- printenv APP_MODE
   # Вивід: prod
   ```
2. Після зміни `APP_MODE: "dev"` у файлі `k8s/configmap.yaml` та виконання `kubectl apply -f k8s/configmap.yaml` повторна перевірка в тому самому pod-і показала старе значення (`prod`), оскільки змінні середовища (`env`) передаються процесу контейнера один раз під час його створення і не оновлюються динамічно в уже запущеному контейнері.
3. Після виконання `kubectl rollout restart deployment k8s-demo-api -n k8s-demo` Kubernetes створив нові pod-и, у яких підхопилося оновлене значення `APP_MODE=dev` (що підтвердилося через `kubectl exec` та `GET /api/status`).

### 2. Аналіз Liveness та Readiness перевірок (`kubectl describe pod`)
* **Liveness Probe:**
  * **Шлях:** `HTTP GET /healthz` (порт `8080`)
  * **Параметри:** `initialDelaySeconds: 5`, `periodSeconds: 10`, `failureThreshold: 3`
* **Readiness Probe:**
  * **Шлях:** `HTTP GET /ready` (порт `8080`)
  * **Параметри:** `initialDelaySeconds: 3`, `periodSeconds: 5`, `failureThreshold: 3`
* **Різниця між перевірками:** `Liveness` перевіряє, чи живий процес застосунку (якщо він завис і не відповідає — kubelet перезапустить контейнер), а `Readiness` перевіряє, чи готовий pod обслуговувати трафік (якщо перевірка падає — pod тимчасово виключається з балансування `Service` без перезапуску контейнера).

---

## Рівень 90 балів: Ingress, Debug та самовідновлення

### 1. Налаштування Ingress та доступ за доменом
* Увімкнено аддон `ingress` у Minikube, застосовано `k8s/ingress.yaml` (заданий `host: k8s-demo.local`), додано запис `127.0.0.1 k8s-demo.local` у файл `hosts` та запущено `minikube tunnel`.
* Перевірка запитів:
  ```powershell
  curl.exe [http://k8s-demo.local/api/status](http://k8s-demo.local/api/status)
  curl.exe [http://k8s-demo.local/api/items](http://k8s-demo.local/api/items)
  ```
* **Шлях запиту:** `Client (curl)` $\rightarrow$ `hosts (127.0.0.1)` $\rightarrow$ `minikube tunnel` $\rightarrow$ `Ingress Controller (ingress-nginx)` $\rightarrow$ правило `Ingress (k8s-demo-ingress)` за `Host: k8s-demo.local` $\rightarrow$ `Service (k8s-demo-api:80)` $\rightarrow$ `Pod (:8080)`.

### 2. Базовий Debug-флоу
Для діагностики роботи сервісів у кластері використовувалися команди:
1. `kubectl describe ingress k8s-demo-ingress -n k8s-demo` — перевіряє правила маршрутизації (Host/Path), призначену IP-адресу та прив'язку до бекенд-сервісу і його ендпоінтів.
2. `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=20` — виводить журнал вхідних HTTP-запитів через Ingress-контролер, коди відповідей (`200`, `404`, `502`) та можливі помилки конфігурації NGINX.
3. `kubectl get events -n k8s-demo --sort-by=.metadata.creationTimestamp` — показує хронологію подій у namespace (створення pod-ів, завантаження образів, спрацювання проб, причини збоїв).

### 3. Демонстрація самовідновлення (Self-healing)
Вивід команд видалення pod-а та автоматичного відновлення репліки:
```text
PS C:\Users\yuriy\source\repos\k8s-demo> kubectl delete pod k8s-demo-api-789f6fc5cf-p497f -n k8s-demo
pod "k8s-demo-api-789f6fc5cf-p497f" deleted from k8s-demo namespace

PS C:\Users\yuriy\source\repos\k8s-demo> kubectl get pods -n k8s-demo
NAME                            READY   STATUS        RESTARTS   AGE
k8s-demo-api-789f6fc5cf-dlbj9   1/1     Running       0          13s
k8s-demo-api-789f6fc5cf-p497f   1/1     Terminating   0          12m
k8s-demo-api-789f6fc5cf-tf5bh   1/1     Running       0          12m
```
**Пояснення:** контролер `ReplicaSet`, яким керує `Deployment`, постійно звіряє фактичну кількість активних pod-ів із заданою (`replicas: 2`). Щойно один pod перейшов у стан `Terminating`, `ReplicaSet` миттєво створив новий pod (`k8s-demo-api-789f6fc5cf-dlbj9`), підтримуючи безперервну доступність сервісу.

---

## Рівень 100 балів: Міграція власного сервісу `AutoService_web` у Kubernetes

### 1. Опис сервісу та контейнеризація
Для міграції обрано власний веб-сервіс **AutoService_web**, призначений для обліку замовлень та послуг станції технічного обслуговування (основні ендпоінти: `GET /health`, `GET /services`, `GET /api/orders/{id}`, `POST /orders`).

Використаний багатоетапний `Dockerfile`:
```dockerfile
FROM [mcr.microsoft.com/dotnet/sdk:9.0](https://mcr.microsoft.com/dotnet/sdk:9.0) AS build
WORKDIR /src

COPY *.csproj ./
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /out

FROM [mcr.microsoft.com/dotnet/aspnet:9.0](https://mcr.microsoft.com/dotnet/aspnet:9.0) AS runtime
WORKDIR /app
COPY --from=build /out ./

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080

ENTRYPOINT ["dotnet", "AutoService_web.dll"]
```

Збірка та локальна перевірка контейнера:
```powershell
docker build -t autoservice-web:1.0.0 .
docker run --rm -p 8081:8080 autoservice-web:1.0.0
curl.exe http://localhost:8081/health
# Відповідь: {"status":"ok"}
curl.exe http://localhost:8081/services
# Відповідь: [{"id":1,"name":"Діагностика"},{"id":2,"name":"Ремонт двигуна"}]
```

### 2. Розгортання в Kubernetes та перевірка конфігурації
Створено маніфести `k8s/autoservice-configmap.yaml`, `k8s/autoservice-deployment.yaml` (2 репліки, `imagePullPolicy: Never`, проби на `/health`, підключення змінних через `envFrom`) та `k8s/autoservice-service.yaml` (`ClusterIP`, порт `80 -> 8080`).

Вивід розгорнутих ресурсів у кластері (`kubectl get deployment,svc,pods -n k8s-demo`):
```text
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/autoservice-deployment   2/2     2            2           11s
deployment.apps/k8s-demo-api             2/2     2            2           26m

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/autoservice-svc   ClusterIP   10.109.234.201   <none>        80/TCP    6s
service/k8s-demo-api      ClusterIP   10.107.239.245   <none>        80/TCP    26m

NAME                                          READY   STATUS    RESTARTS   AGE
pod/autoservice-deployment-58bdd95856-tccnc   1/1     Running   0          11s
pod/autoservice-deployment-58bdd95856-tcw82   1/1     Running   0          11s
```

Перевірка роботи сервісу з кластера через `port-forward` (`8082:80`) та оновлення `ConfigMap` (`SERVICE_MODE: "standard"` $\rightarrow$ `"high-load"`):
```text
PS C:\Users\yuriy\source\repos\AutoService_web> curl.exe http://localhost:8082/health
{"status":"ok"}

PS C:\Users\yuriy\source\repos\k8s-demo> kubectl rollout restart deployment autoservice-deployment -n k8s-demo
deployment.apps/autoservice-deployment restarted
PS C:\Users\yuriy\source\repos\k8s-demo> kubectl exec -n k8s-demo autoservice-deployment-c4ff6bcfc-76h52 -- printenv SERVICE_MODE
high-load
```

---

## Відповіді на контрольні запитання

1. **У чому відмінність між `ConfigMap` та `Secret`?**
   * `ConfigMap` призначений для зберігання звичайних неконфіденційних параметрів конфігурації у відкритому вигляді (режим роботи, URL-адреси, прапорці функцій). `Secret` використовується для чутливих даних (паролі, API-токени, ключі шифрування), які кодуються в Base64, можуть обмежуватися правами доступу (RBAC) та монтуються в пам'ять (`tmpfs`), не потрапляючи у відкритий текст маніфестів Deployment.

2. **У чому відмінність між `liveness`- та `readiness`-перевірками?**
   * `Liveness probe` визначає, чи не завис процес усередині контейнера: якщо перевірка провалюється, Kubernetes перезапускає контейнер. `Readiness probe` визначає, чи готовий застосунок приймати запити (наприклад, після прогріву чи підключення до БД): якщо перевірка провалюється, Kubernetes прибирає IP цього pod-а зі списку ендпоінтів `Service`, щоб клієнти не отримували помилки, але сам контейнер не перезапускає.

3. **Через які ресурси проходить запит при зверненні:**
   * **i. Через `kubectl port-forward`:** локальний порт ПК (`localhost:8080` / `8082`) $\rightarrow$ тунель через Kubernetes API Server (kubelet) $\rightarrow$ `Service` (`ClusterIP`) $\rightarrow$ один із цільових `Pod`-ів $\rightarrow$ порт контейнера (`8080`).
   * **ii. Через `http://k8s-demo.local`:** клієнт (`curl`/браузер) $\rightarrow$ файл `hosts` (`127.0.0.1`) $\rightarrow$ `minikube tunnel` $\rightarrow$ контролер `Ingress` (`ingress-nginx-controller`) $\rightarrow$ об'єкт `Ingress` (`k8s-demo-ingress`, маршрутизація за `Host` і `Path`) $\rightarrow$ `Service` (`k8s-demo-api`, `ClusterIP`) $\rightarrow$ `Pod` (`8080`).

4. **Як рекомендується оновлювати конфігурацію, якщо вона надходить через env-змінні з `ConfigMap`?**
   * Оскільки змінні середовища фіксуються при старті контейнера, після оновлення `ConfigMap` (`kubectl apply`) необхідно виконати плановий перезапуск pod-ів через `kubectl rollout restart deployment <name>` (що забезпечує оновлення без простою — Rolling Update). У продакшн-середовищах для автоматизації цього процесу використовують інструменти на кшталт `Reloader` або додають у шаблон pod-а анотацію з хешем вмісту `ConfigMap`.

5. **Як було мігровано власний сервіс у Kubernetes і які відмінності від демо `k8s-demo`?**
   * Для сервісу `AutoService_web` створено багатоетапний `Dockerfile` (збірка на `dotnet/sdk:9.0` та запуск на легкому `dotnet/aspnet:9.0`), зібрано локальний образ, завантажено в Minikube (`minikube image load`) та описано маніфести `ConfigMap`, `Deployment` (`imagePullPolicy: Never`, 2 репліки) і `Service` (`ClusterIP`). Головні відмінності від Node.js-демо `k8s-demo`: використання багатоетапної компіляції C# замість інтерпретованого JS, налаштування `ASPNETCORE_URLS=http://+:8080` для прослуховування порту в контейнері, використання єдиного ендпоінту `/health` для обох проб (`liveness` та `readiness`) і вбудована база SQLite всередині контейнера.
