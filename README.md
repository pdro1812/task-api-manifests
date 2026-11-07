# task-api-manifests: estado desejado (GitOps)

Repositório de manifestos Kubernetes da Task API. O Argo CD, dentro do cluster, sincroniza o cluster com o conteúdo
deste repositório. A CI do repositório da aplicação (`task-api`) só grava aqui a nova tag de imagem e não acessa o
cluster.

**Este repositório não é executado sozinho.** Ele é usado pelo `experiments/setup.sh` do repositório `task-api`, e os
dois devem ser clonados lado a lado (`task-api/` e `task-api-manifests/`). As instruções completas de reprodução estão
no README de `task-api`.

| Arquivo | Conteúdo |
|---|---|
| `1-redis.yaml` | Service, PVC (1 Gi, AOF) e Deployment do Redis 7 (estratégia `Recreate`; probes autenticadas; imagem por digest) |
| `2-config.yaml` | ConfigMap da API (porta, host e porta do Redis) |
| `2-sealed-secret.yaml` | senha do Redis selada (`SealedSecret`), decifrável apenas pelo controller do cluster original |
| `3-api.yaml` | Service e Deployment da API (2 réplicas, `maxUnavailable: 0`, readiness/liveness, `preStop` de 5 s); a tag de imagem é gravada pela CI |
| `argocd-app.yaml` | `Application` do Argo CD que sincroniza este repositório (sync automático com `prune` e `selfHeal`) |
| `argocd-sealed-secrets.yaml`, `sealed-secrets/` | `Application` e manifesto versionado do controller Sealed Secrets v0.41.2 (imagem por digest) |

## Uso

- **Sem GitOps** (cenários C1, C2, C4 e C5 imperativo): o `setup.sh` do `task-api` aplica `1-redis.yaml`,
  `2-config.yaml` e `3-api.yaml` com `kubectl apply` e cria uma senha local para o Redis, porque o `SealedSecret`
  não decifra num cluster novo.
- **Com GitOps** (C3 e C5 GitOps): faça um fork deste repositório, troque `repoURL` em `argocd-app.yaml` e
  `argocd-sealed-secrets.yaml` pela URL do fork e rode `GITOPS=1 ./experiments/setup.sh` no `task-api`. O fork é
  necessário porque os experimentos fazem commits e pushes aqui, e o Argo CD lê as mudanças de uma URL Git.
- **Outra senha do Redis**: sele um novo `Secret` com `kubeseal` usando o certificado do controller do seu cluster e
  substitua `2-sealed-secret.yaml`. Depois, incremente a anotação `task-api/redis-secret-rev` em `3-api.yaml`, para
  que a API reinicie com a senha nova.
