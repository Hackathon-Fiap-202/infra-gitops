# infra-gitops

Repositório GitOps contendo a infraestrutura e configurações de deploy automatizado para o projeto Nextime, utilizando ArgoCD para gerenciar aplicações Kubernetes no cluster EKS da AWS.

## 📋 Sobre o Projeto

Este repositório implementa uma estratégia GitOps para gerenciar o ciclo de vida das aplicações em Kubernetes. Utiliza ArgoCD como ferramenta de Continuous Delivery, permitindo que mudanças no repositório sejam automaticamente sincronizadas com o cluster Kubernetes.

### Conceitos GitOps

- **Fonte única da verdade**: Todas as configurações de infraestrutura estão versionadas neste repositório
- **Sincronização automática**: ArgoCD monitora este repositório e aplica mudanças automaticamente
- **Self-healing**: O ArgoCD garante que o estado desejado seja mantido, revertendo mudanças manuais
- **Auditoria completa**: Todo histórico de mudanças está no Git

## 🏗️ Arquitetura

```
┌─────────────────┐
│   GitHub Repo   │ (Este repositório)
│   infra-gitops  │
└────────┬────────┘
         │
         │ Monitora mudanças
         ▼
┌─────────────────┐
│     ArgoCD      │ (Instalado no cluster)
│   (Controller)  │
└────────┬────────┘
         │
         │ Aplica manifests
         ▼
┌─────────────────┐
│   EKS Cluster   │
│  nextime-cluster│
│   (AWS us-east-1)│
└─────────────────┘
```

## 📁 Estrutura do Repositório

```
infra-gitops/
├── argocd/
│   ├── applications/          # Definições das aplicações ArgoCD
│   │   ├── ms-order.yaml      # Aplicação ms-video
│   │   └── ms-process-video.yaml
│   └── apps-ingress/          # Configuração de Ingress
│       └── ingress.yaml       # AWS ALB Ingress Controller
├── .github/
│   └── workflows/
│       └── validate.yml       # CI/CD Pipeline (GitHub Actions)
└── README.md
```

## 🚀 Aplicações Gerenciadas

### 1. ms-video
- **Repositório**: `https://github.com/Hackathon-Fiap-202/ms-video`
- **Branch**: `main`
- **Path**: `infra/k8s`
- **Namespace**: `default`
- **Sincronização**: Automática com self-healing

### 2. ms-process-video
- **Repositório**: `https://github.com/Hackathon-Fiap-202/ms-process-video`
- **Branch**: `main`
- **Path**: `infra/k8s`
- **Namespace**: `default`
- **Sincronização**: Automática com self-healing

## 🌐 Ingress e Roteamento

O Ingress configurado utiliza o **AWS Application Load Balancer (ALB)** para rotear o tráfego:

- **Ingress Class**: `alb`
- **Scheme**: `internet-facing`
- **Target Type**: `ip`
- **Group**: `nextime-infra-group`

### Rotas Configuradas

| Path | Service | Port |
|------|---------|------|
| `/video` | `ms-video` | 80 |
| `/proccess-video` | `ms-proccess-video` | 80 |

## ⚙️ Pré-requisitos

- **Kubernetes Cluster**: EKS na AWS (nextime-cluster)
- **ArgoCD**: Instalado e configurado no cluster
- **AWS CLI**: Configurado com credenciais apropriadas
- **kubectl**: Configurado para acessar o cluster EKS
- **AWS ALB Ingress Controller**: Instalado no cluster

## 🔧 Configuração

### 1. Configurar Acesso ao Cluster EKS

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name nextime-cluster
```

### 2. Verificar Instalação do ArgoCD

```bash
kubectl get pods -n argocd
```

### 3. Aplicar Configurações Manualmente (se necessário)

```bash
kubectl apply -R -f argocd/
```

## 🔄 CI/CD Pipeline

O repositório possui um pipeline GitHub Actions (`.github/workflows/validate.yml`) que:

1. **Trigger**: Executa em push e pull requests para a branch `main`
2. **Autenticação**: Utiliza OIDC para autenticação segura com AWS
3. **Deploy**: Aplica automaticamente os manifests do ArgoCD
4. **Validação**: Verifica se os pods das aplicações estão prontos

### Secrets Necessários no GitHub

- `AWS_ACCOUNT_ID`: ID da conta AWS

### IAM Role Necessária

- `arn:aws:iam::<ACCOUNT_ID>:role/github-action-role`

## 📝 Como Adicionar uma Nova Aplicação

1. Crie um novo arquivo em `argocd/applications/`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nome-da-aplicacao
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo
    targetRevision: main
    path: infra/k8s
    kustomize: {}
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

2. Adicione a rota no `argocd/apps-ingress/ingress.yaml` se necessário
3. Faça commit e push para a branch `main`
4. O ArgoCD sincronizará automaticamente a nova aplicação

## 🔍 Monitoramento e Troubleshooting

### Verificar Status das Aplicações no ArgoCD

```bash
kubectl get applications -n argocd
```

### Ver Logs do ArgoCD

```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### Verificar Status dos Pods

```bash
kubectl get pods -n default
kubectl get pods -l app=ms-video -n default
kubectl get pods -l app=ms-proccess-video -n default
```

### Verificar Ingress

```bash
kubectl get ingress -n default
kubectl describe ingress apps-ingress -n default
```

### Sincronização Manual (se necessário)

```bash
argocd app sync ms-video
argocd app sync ms-process-video
```

## 🔐 Segurança

- **OIDC**: Autenticação sem necessidade de armazenar credenciais AWS
- **RBAC**: Controle de acesso baseado em roles do Kubernetes
- **Secrets**: Informações sensíveis não são versionadas no repositório
- **Network Policies**: (Recomendado implementar para isolamento de rede)

## 📚 Recursos Adicionais

- [Documentação ArgoCD](https://argo-cd.readthedocs.io/)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [AWS ALB Ingress Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [GitOps Principles](https://www.gitops.tech/)

## 👥 Contribuindo

1. Faça um fork do repositório
2. Crie uma branch para sua feature (`git checkout -b feature/nova-aplicacao`)
3. Commit suas mudanças (`git commit -m 'Adiciona nova aplicação'`)
4. Push para a branch (`git push origin feature/nova-aplicacao`)
5. Abra um Pull Request

## 📄 Licença

Este projeto faz parte do Hackathon FIAP 2026.

---

**Nota**: Este repositório é a fonte única da verdade para a infraestrutura. Mudanças manuais no cluster serão revertidas pelo ArgoCD se não estiverem refletidas aqui.
