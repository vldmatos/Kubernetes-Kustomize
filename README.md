# Kubernetes + Kustomize: NGINX por ambiente

Este repositório demonstra como organizar manifests do Kubernetes usando Kustomize para customizar um mesmo app (NGINX) para diferentes ambientes: development, staging e production.

Principais diferenças por ambiente:
- Réplicas do Deployment: dev=1, stg=2, prod=3
- Página HTML (index.html) específica via ConfigMapGenerator
- SecretGenerator com variáveis de ambiente por ambiente

Estrutura:

```text
application/
  kustomization.yaml
  deployment.yaml
  service.yaml
environments/
  development/
    kustomization.yaml
    index.html
    ingress.yaml
    patch-resources.yaml
  staging/
    kustomization.yaml
    index.html
    ingress.yaml
    patch-resources.yaml
  production/
    kustomization.yaml
    index.html
    ingress.yaml
    patch-resources.yaml
```

## Como aplicar

Requer kubectl com suporte a `-k` (Kustomize embutido):

```pwsh
# Development
kubectl apply -k environments/development

# Staging
kubectl apply -k environments/staging

# Production
kubectl apply -k environments/production
```

Para apenas inspecionar os manifests renderizados:

```pwsh
kubectl kustomize environments/development
```

## O que é gerado

- Deployment `nginx-app` (image: `nginx:1.25-alpine`), com volume do ConfigMap `web-content` montado em `/usr/share/nginx/html`.
- Service `nginx-app` (ClusterIP:80).
- Ingress por ambiente (hosts: `nginx-dev.local`, `nginx-stg.local`, `nginx-prod.local`, classe `nginx`).
- ConfigMap por ambiente contendo `index.html` (configMapGenerator com `disableNameSuffixHash: true`).
- Secret por ambiente com variáveis `APP_ENV`, `APP_USER`, `APP_TOKEN` (secretGenerator com `disableNameSuffixHash: true`).
- Patch de resources por ambiente (CPU/Mem requests/limits) via `patchesStrategicMerge`.

Notas:

- O Deployment de base define `replicas: 1`, mas os overlays substituem via campo `replicas` do Kustomize.
- Ajuste as `resources` (requests/limits) conforme sua capacidade de cluster.
- Para evitar expor o token em repositório, considere substituir `secretGenerator.literals` por `envs` (arquivo .env específico por ambiente) e/ou integrar com `sealed-secrets`/`ExternalSecrets`.

### Dicas de Ingress

- Certifique-se de possuir um controlador de Ingress (por exemplo, ingress-nginx) instalado e que a `ingressClassName: nginx` exista.
- Adicione entradas no seu DNS/hosts apontando para o IP do Ingress Controller:
  - nginx-dev.local, nginx-stg.local, nginx-prod.local
