# ToggleMaster 🚀

Configurações de deployment e orquestração de containers para a plataforma **ToggleMaster** usando ArgoCD e Kubernetes.

## 🎯 Descrição do Serviço

Este diretório contém as configurações declarativas para deployment de todos os serviços ToggleMaster em Kubernetes. Ele:

1. Utiliza ArgoCD para GitOps (declarar estado desejado em Git)
2. Define Helm charts para cada serviço
3. Gerencia configurações por ambiente (dev, staging, production)
4. Orquestra updates automáticos baseados em Git
5. Fornece rollback automático em caso de falhas

**Função crítica:** Este é o "coração" da orquestração. Sem ele, não há deployment automático dos serviços.

## 📦 Stack Técnico

- **Orquestração:** Kubernetes 1.24+
- **GitOps:** ArgoCD
- **Package Manager:** Helm 3+
- **Container Registry:** Docker Hub / ECR / GCR
- **Config Management:** ConfigMap, Secrets

## 🗂️ Estrutura de Diretórios

```
togglemaster/
├── argocd/                     # Configurações ArgoCD
│   ├── bootstrap-production.yaml
│   ├── bootstrap-staging.yaml
│   ├── bootstrap.yaml
│   └── applications/           # ApplicationSets para cada ambiente
├── helm/                       # Helm charts
│   ├── analytics-service/
│   ├── auth-service/
│   ├── evaluation-service/
│   ├── flag-service/
│   ├── targeting-service/
│   └── togglemaster-chart/     # Chart principal
├── kustomize/                  # Overlays Kustomize (opcional)
│   ├── base/
│   ├── dev/
│   ├── staging/
│   └── production/
└── README.md
```

## 🚀 Como Usar

### Pré-requisitos Locais

- Kubernetes cluster (EKS, AKS, GKE ou Minikube)
- kubectl configurado
- Helm 3+ instalado
- ArgoCD CLI (opcional, para testes)
- Git (para GitOps)

### Setup Inicial

#### 1. Clonar o Repositório
```bash
cd togglemaster
```

#### 2. Instalar ArgoCD no Cluster

```bash
# Criar namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Aguarde os pods iniciarem
kubectl wait -n argocd --for=condition=ready pod -l app.kubernetes.io/name=argocd-server --timeout=300s
```

#### 3. Acessar ArgoCD UI

```bash
# Port forward para acesso local
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Obter senha de admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Acessar em https://localhost:8080
```

#### 4. Configurar Repositório Git

```bash
# Login no ArgoCD
argocd login localhost:8080 --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# Adicionar repositório
argocd repo add https://github.com/seu-usuario/togglemaster.git \
  --username git-user \
  --password git-token
```

#### 5. Criar Aplicação ArgoCD

```bash
# Aplicar bootstrap (production)
kubectl apply -f argocd/bootstrap-production.yaml

# Ou via ArgoCD CLI
argocd app create togglemaster \
  --repo https://github.com/seu-usuario/togglemaster.git \
  --path togglemaster/helm \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

#### 6. Sincronizar Aplicações

```bash
# Sincronizar via CLI
argocd app sync togglemaster

# Ou via UI em https://localhost:8080
```

### Testando Localmente com Minikube

#### 1. Inicie Minikube
```bash
minikube start --cpus=4 --memory=8192
```

#### 2. Ative o Docker registry local
```bash
eval $(minikube docker-env)
```

#### 3. Build das imagens
```bash
docker build -t togglemaster/auth-service:latest ../Auth-Service
docker build -t togglemaster/flag-service:latest ../Flag-Service
docker build -t togglemaster/evaluation-service:latest ../Evaluation-Service
docker build -t togglemaster/targeting-service:latest ../Targeting-Service
docker build -t togglemaster/analytics-service:latest ../Analytics-Service
```

#### 4. Deploy via Helm
```bash
# Para development
helm install togglemaster ./helm -f helm/values-dev.yaml

# Para testar alterações
helm upgrade togglemaster ./helm -f helm/values-dev.yaml
```

#### 5. Verificar Deployment
```bash
kubectl get pods
kubectl get svc
kubectl logs deployment/auth-service
```

## 🔧 Variáveis de Ambiente / Secrets

### GitHub Secrets Necessários

```yaml
KUBECONFIG
  Descrição: Arquivo de configuração Kubernetes (base64)
  Valor: <base64-encoded-kubeconfig>

DOCKER_REGISTRY_URL
  Descrição: URL do registry Docker
  Valor: docker.io

DOCKER_REGISTRY_USERNAME
  Descrição: Username do registry Docker
  Valor: <seu-username>

DOCKER_REGISTRY_PASSWORD
  Descrição: Token do registry Docker
  Valor: <seu-token>

ARGOCD_SERVER
  Descrição: URL do servidor ArgoCD
  Valor: argocd.example.com

ARGOCD_AUTH_TOKEN
  Descrição: Token de autenticação ArgoCD
  Valor: <seu-token>

GIT_REPOSITORY_URL
  Descrição: URL do repositório Git
  Valor: https://github.com/seu-usuario/togglemaster.git

GIT_REPOSITORY_USERNAME
  Descrição: Username para Git
  Valor: <seu-username>

GIT_REPOSITORY_TOKEN
  Descrição: Token pessoal GitHub
  Valor: <seu-token>

KARPENTER_ROLE_ARN
  Descrição: ARN da role IAM para Karpenter (auto-scaling)
  Valor: arn:aws:iam::123456789012:role/karpenter-role

DATADOG_API_KEY
  Descrição: Chave API Datadog
  Valor: <sua-api-key>

HELM_REPOSITORY_PASSWORD
  Descrição: Senha do Helm repository (se privado)
  Valor: <sua-senha>
```

## 📊 Helm Charts

### Estrutura de um Chart

```
helm/analytics-service/
├── Chart.yaml              # Metadata do chart
├── values.yaml             # Valores padrão
├── values-dev.yaml         # Overrides para dev
├── values-staging.yaml     # Overrides para staging
├── values-production.yaml  # Overrides para production
└── templates/
    ├── deployment.yaml     # Definição do Deployment
    ├── service.yaml        # Serviço Kubernetes
    ├── configmap.yaml      # ConfigMap
    ├── secrets.yaml        # Secrets (encrypted)
    ├── hpa.yaml           # Horizontal Pod Autoscaler
    └── ingress.yaml       # Ingress
```

### Instalar Chart Específico

```bash
# Listar charts
helm list

# Instalar nova release
helm install analytics-service ./helm/analytics-service \
  -f ./helm/analytics-service/values-production.yaml

# Upgrade de release existente
helm upgrade analytics-service ./helm/analytics-service \
  -f ./helm/analytics-service/values-production.yaml

# Desinstalar release
helm uninstall analytics-service

# Testar sem aplicar
helm template analytics-service ./helm/analytics-service \
  -f ./helm/analytics-service/values-production.yaml
```

## 🔐 GitHub Secrets Necessários (Resumo)

Configure TODOS estes secrets para CI/CD funcionar:

```yaml
# Kubernetes
KUBECONFIG (base64 encoded)
KARPENTER_ROLE_ARN

# Docker Registry
DOCKER_REGISTRY_URL
DOCKER_REGISTRY_USERNAME
DOCKER_REGISTRY_PASSWORD

# Git
GIT_REPOSITORY_URL
GIT_REPOSITORY_USERNAME
GIT_REPOSITORY_TOKEN

# ArgoCD
ARGOCD_SERVER
ARGOCD_AUTH_TOKEN

# Monitoring
DATADOG_API_KEY

# Helm (se privado)
HELM_REPOSITORY_PASSWORD
```

## 📈 Deployment Workflow

```
1. Desenvolvedor faz commit para Git
   ↓
2. GitHub Actions (CI) executa
   - Build das imagens Docker
   - Push para registry
   - Atualiza values.yaml
   ↓
3. ArgoCD detecta mudança no Git
   ↓
4. ArgoCD sincroniza com cluster
   - Cria/atualiza Deployments
   - Rola novos Pods
   - Monitora saúde
   ↓
5. Aplicação em produção!
```

## 🔄 GitOps com ArgoCD

### Estrutura de Aplicações

```yaml
# argocd/bootstrap-production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: togglemaster-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/seu-usuario/togglemaster.git
    targetRevision: main
    path: togglemaster/helm
  destination:
    server: https://kubernetes.default.svc
    namespace: togglemaster-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Sincronizar Manualmente

```bash
# Via CLI
argocd app sync togglemaster-production

# Forçar sincronização
argocd app sync togglemaster-production --force

# Aguardar conclusão
argocd app wait togglemaster-production
```

## 🐛 Troubleshooting

### Problema: "ApplicationSet stuck in pending"
**Solução:** Verifique credenciais Git
```bash
kubectl describe applicationset togglemaster -n argocd
```

### Problema: "ImagePullBackOff"
**Solução:** Verifique credenciais do registry Docker
```bash
kubectl logs pod-name
kubectl describe pod pod-name
```

### Problema: "CrashLoopBackOff"
**Solução:** Verifique logs da aplicação
```bash
kubectl logs deployment/auth-service -n togglemaster-prod
```

### Problema: ArgoCD não sincroniza mudanças
**Solução:** Force sync manual
```bash
argocd app sync togglemaster --force
```

## 📊 Monitoramento de Deployments

### Status dos Aplicativos
```bash
# Ver status
kubectl get applications -n argocd

# Detalhes de uma aplicação
kubectl describe app togglemaster-production -n argocd

# Histórico de syncs
argocd app history togglemaster-production
```

### Pod Status
```bash
# Listar todos os pods
kubectl get pods -n togglemaster-prod

# Ver detalhes
kubectl describe pod <pod-name> -n togglemaster-prod

# Logs
kubectl logs <pod-name> -n togglemaster-prod

# Shell interativo
kubectl exec -it <pod-name> -n togglemaster-prod -- /bin/sh
```

## 🔄 Rollback de Deployments

### Via ArgoCD

```bash
# Ver histórico
argocd app history togglemaster-production

# Rollback para revision anterior
argocd app rollback togglemaster-production 1

# Rollback para revision específica
argocd app rollback togglemaster-production <revision>
```

### Via Helm

```bash
# Ver releases
helm history togglemaster

# Rollback para release anterior
helm rollback togglemaster

# Rollback para release específica
helm rollback togglemaster <revision>
```

## 📚 Boas Práticas

### GitOps
1. **Sempre declare estado desejado em Git**
2. **Use branches para ambientes diferentes**
3. **Implemente code review antes de merge**
4. **Mantenha secrets fora do Git (use Sealed Secrets)**

### Kubernetes
1. **Use namespaces para isolamento**
2. **Configure requests/limits de recursos**
3. **Implemente health checks (readiness/liveness)**
4. **Use network policies para segurança**

### ArgoCD
1. **Enable auto-sync para ambientes não-críticos**
2. **Disable auto-sync para produção (requer aprovação manual)**
3. **Configure webhook para updates em tempo real**
4. **Use RBAC para controle de acesso**

## 📚 Recursos Adicionais

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GitOps Best Practices](https://www.weave.works/technologies/gitops/)
- [ToggleMaster Architecture](../README.md)

## 🔗 Links Importantes

- ArgoCD Dashboard: https://argocd.example.com
- Helm Hub: https://artifacthub.io/
- Kubernetes Docs: https://kubernetes.io/docs/

## 👥 Suporte

Para dúvidas ou problemas com deployment, abra uma issue no repositório ou entre em contato com o time DevOps.

## ⚠️ AVISO IMPORTANTE

**Nunca faça push de secrets em plaintext para Git!**

Sempre use:
- GitHub Secrets para CI/CD
- Sealed Secrets ou SOPS para Kubernetes secrets
- HashiCorp Vault para produção

A segurança é responsabilidade de todos!
