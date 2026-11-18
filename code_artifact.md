# Інструкція із запуску та перевірки Kubernetes-стенду

Перед початком роботи переконайтеся, що **Docker Desktop** запущено.

## 1. Запуск кластера та перевірка ресурсів
*(Виконується у звичайному терміналі Visual Studio / PowerShell)*

```powershell
# Перехід у директорію репозиторію з маніфестами
cd C:\Users\yuriy\source\repos\k8s-demo

# Запуск локального кластера Minikube (відновлює всі збережені образи та налаштування)
minikube start

# Перевірка статусу Deployment, Service та Pod-ів (мають бути 4/4 Running)
kubectl get deployment,svc,pods -n k8s-demo

# Перевірка активності Ingress-контролера (ingress-nginx-controller має бути 1/1 Running)
kubectl get pods -n ingress-nginx

# (За потреби) Повторне застосування всіх маніфестів, якщо ресурси були видалені
kubectl apply -f k8s/
```

## 2. Відкриття мережевих тунелів
Для одночасної роботи обох сервісів потрібно запустити два тунелі в окремих вікнах терміналу.

**Вікно 1 — PowerShell від імені Адміністратора** *(для доступу до `http://k8s-demo.local`)*:
```powershell
# Створення мережевого маршруту між 127.0.0.1:80 та Ingress-контролером (не закривати вікно під час роботи)
minikube tunnel
```

**Вікно 2 — Термінал Visual Studio** *(для доступу до `AutoService_web`)*:
```powershell
# Прокидання локального порту 8082 на порт 80 сервісу autoservice-svc (не закривати вікно під час роботи)
kubectl port-forward svc/autoservice-svc -n k8s-demo 8082:80
```

## 3. Перевірка роботи обох сервісів
*(Виконується у новому вікні терміналу або через браузер)*

```powershell
# Перевірка статусу навчального сервісу через Ingress (домен k8s-demo.local)
curl.exe http://k8s-demo.local/api/status

# Перевірка отримання списку елементів із навчального сервісу
curl.exe http://k8s-demo.local/api/items

# Перевірка працездатності власного .NET 9 сервісу AutoService_web
curl.exe http://localhost:8082/health

# Перевірка отримання списку послуг СТО з бази даних AutoService_web
curl.exe http://localhost:8082/services
```

## 4. Коректне завершення роботи

```powershell
# 1. В обох вікнах із тунелями (minikube tunnel та kubectl port-forward) натисніть Ctrl + C

# 2. Зупинка кластера Minikube для звільнення оперативної пам'яті ПК
minikube stop
```