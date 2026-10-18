# Runbooks и упражнения по эксплуатации

## Порядок запуска (bootstrap)
1. Закоммить `argocd-apps/`, `apps/` в `k8s-gitops`, заменить `YOUR_GH_USER`, диапазон IP в `metallb.yaml`, адрес Vault в `secret-store.yaml`.
2. root-app подхватит platform apps (waves: 0 операторы -> 1 traefik/мониторинг -> 2 platform-config -> 3 auth-app).
3. Vault: `vault kv put secret/auth-app/jwt secret=$(openssl rand -hex 32)`; роль `external-secrets` (Kubernetes auth) + read-политика.
4. Репо `auth-app`: secret `GITOPS_TOKEN` (fine-grained PAT, write на `k8s-gitops`). Push в main -> образ -> коммит тега -> в ArgoCD Refresh + Sync.
5. /etc/hosts: `<IP Traefik от MetalLB> auth.home.lab`; корневой CA: `kubectl -n cert-manager get secret homelab-ca -o jsonpath='{.data.ca\.crt}' | base64 -d`.

## high-error-rate
1. Grafana: RED-дашборд, по `path`/`status` найти источник. 2. `kubectl -n auth-app get pods,events`; `logs` (или Loki: `{namespace="auth-app"} | json | status>=500`).
3. Был деплой? Откат = `git revert` коммита с тегом в k8s-gitops -> Sync. НЕ `kubectl rollout undo` (ArgoCD вернёт как в Git).
4. БД: `kubectl cnpg status auth-db`.

## Упражнения (ради этого всё и затевалось)
| # | Сценарий | Что наблюдаем | Чему учит |
|---|----------|---------------|-----------|
| 1 | `kubectl drain worker2 --ignore-daemonsets --delete-emptydir-data` | PDB держит >=2 пода; CNPG переключает primary | PDB, anti-affinity |
| 2 | `kubectl delete pod auth-db-1` (primary) | failover на реплику за секунды, readiness app мигает | HA БД, поведение pool |
| 3 | Жёстко выключить воркер | время до эвакуации подов (~5 мин tolerations) | node failure, tolerationSeconds |
| 4 | Плохой образ (CMD падает) в main | startup/readiness не пускают трафик, старые поды живы | rolling update, rollback через Git |
| 5 | Утечка памяти / `limits.memory` слишком малый | OOMKilled, алерт, правка в Git | requests/limits |
| 6 | `kubectl scale` руками | ArgoCD показывает OutOfSync/self-heal | drift |
| 7 | Нагрузка `hey -z 2m -c 50 https://auth.home.lab/` | HPA 3->6, латентность p99 | HPA, capacity |
| 8 | Скрипт перебора паролей | алерт AuthBruteForce | бизнес-алертинг |
| 9 | Удалить namespace auth-app целиком | восстановление из Git + бэкап БД | DR |
| 10 | Восстановление БД из бэкапа в новый кластер (фаза 3) | PITR | бэкап, не проверенный восстановлением, — не бэкап |
