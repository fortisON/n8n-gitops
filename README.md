# n8n on Kubernetes (GitOps / FluxCD)

Фінальний проєкт курсу. Розгортання n8n у Kubernetes через FluxCD.
Flux сам котить зміни з цього репо в кластер, без `kubectl apply`.

База даних — PostgreSQL, піднімає оператор CloudNativePG (тобто НЕ plain
StatefulSet/Deployment, а через Custom Resource). TLS — self-signed через
cert-manager.

## Стек

- n8n — офіційний образ `n8nio/n8n`, власний helm-чарт у `charts/n8n`
- PostgreSQL — оператор CloudNativePG (CR `Cluster`)
- cert-manager — self-signed сертифікати (ClusterIssuer)
- FluxCD — GitOps
- Traefik — ingress (вбудований у k3s/Rancher Desktop)

## Структура

```
charts/n8n           власний helm-чарт (image/tag/replicas/resources/ingress/hpa)
infrastructure/      оператори (cloudnative-pg, cert-manager) + self-signed ClusterIssuer
apps/staging         ns staging: 1 репліка, мін. ресурси, PG 1 інстанс
apps/production      ns production: HPA 2-5, requests/limits, PG HA (2 інстанси)
clusters/my-cluster  точки входу Flux (Kustomizations + flux-system)
```

## Середовища

- staging: ns `staging`, 1 под, домен `n8n.staging.local`
- production: ns `production`, HPA 2-5 подів, домен `n8n.local`

## Деплой

```
export GITHUB_TOKEN=...
flux bootstrap github --owner=fortisON --repository=n8n-gitops --branch=main --path=./clusters/my-cluster --personal
```

Далі Flux сам усе підніме (оператори -> бази -> n8n). Дивитись прогрес:
`flux get kustomizations -A`.

## Доступ

У `/etc/hosts`:

```
127.0.0.1 n8n.local n8n.staging.local
```

Відкрити https://n8n.local (сертифікат self-signed, браузер лається — це норм).

## Перевірка

```
flux get helmreleases -A
flux get kustomizations -A
kubectl get pods -A
kubectl get ingress -A
```

(виводи додам після деплою)

## Нотатки

- Ключ шифрування n8n лежить у репо відкритим — лише для стенда, не для прод.
- Пароль до PostgreSQL генерує сам оператор (secret `n8n-db-app`), у git його нема.
- Образ не білдимо, тому CI у `.github/workflows` лише валідує маніфести.
