# Hands-on e exercícios

Os hands-on e exercícios da disciplina foram aplicados **direto no projeto do grupo**, em vez de num
repositório de estudos separado: cada um virou uma parte da plataforma. As tabelas mostram onde cada
atividade está, quem fez e onde está o código.

## Hands-on

| Hands-on | Tema | Onde está no projeto | Quem |
|---|---|---|---|
| 0 | Pré-requisitos, repositório raiz e MkDocs | [`pna26/platform`](https://github.com/pna26/platform){target="_blank"} com os serviços como submódulos em `api/` | Leonardo |
| 1 | Account (controller, service, repository, container) | [Account](projeto/account.md): [`account`](https://github.com/pna26/account){target="_blank"} + [`account-service`](https://github.com/pna26/account-service){target="_blank"} | **eu** |
| 2 | Gateway | [`gateway-service`](https://github.com/pna26/gateway-service){target="_blank"} | Leonardo |
| 3 | Segurança: JWT, rotas abertas, autorização | [Auth](projeto/auth.md) (JWT, login, `/auth/solve`, rota `POST /accounts/login` no Account); filtro de autorização no gateway | **eu** (Auth) e Leonardo (gateway) |
| 4 | CI/CD | GitHub Actions no lugar do Jenkins: pipeline reutilizável do `platform`. Eu fiz a parte da AWS: role OIDC e `id-token: write` nos meus serviços ([AWS](projeto/aws.md)) | Leonardo (pipeline) e **eu** (AWS) |
| 5 | Orquestração (Kubernetes) | `k8s/k8s.yaml` do account e do auth, testados no Kubernetes do Docker Desktop e depois no EKS ([EKS](projeto/eks.md#manifestos-dos-meus-servicos)); manifestos do gateway e do `platform` | **eu** (account/auth, EKS) e Leonardo (gateway, lb, Redis) |
| 6 | Observabilidade (Prometheus + Grafana) | [Observabilidade](https://pna26.github.io/platform/bottlenecks/observability/){target="_blank"} | Gabriel |
| 7 | Balanceamento de carga (Nginx) | [Balanceamento](https://pna26.github.io/platform/bottlenecks/load-balancing/){target="_blank"} | Leonardo |
| 8 | Cache distribuído (Redis) | [Cache do token](https://pna26.github.io/platform/bottlenecks/cache/){target="_blank"} | Leonardo |
| 9 | Replicação e failover do banco | não aplicado: no EKS o banco é o RDS, gerenciado pela AWS | — |
| 10 | Mensageria (Kafka) | [Mensageria](bottlenecks.md#mensageria-kafka): outbox no account-service e o [`notification-service`](https://github.com/pna26/notification-service){target="_blank"} | **eu** |
| 11 | Rastreamento distribuído | não aplicado no projeto | — |
| 12 | IA no CI/CD | não aplicado no projeto (a IA foi usada no desenvolvimento, ver [Uso de IA](uso-de-ia.md)) | — |

## Exercícios

| Exercício | Situação | Quem |
|---|---|---|
| [Exchange API](https://insper.github.io/platform/2026.2/exercises/exchange/){target="_blank"} | feito em Python/FastAPI: [`exchange-service`](https://github.com/pna26/exchange-service){target="_blank"} | Gabriel |
| [Product API](https://insper.github.io/platform/2026.2/exercises/product/){target="_blank"} e [Order API](https://insper.github.io/platform/2026.2/exercises/order/){target="_blank"} | não feitos: o escopo do projeto nesta edição são os quatro serviços Gateway, Auth, Account e Exchange | — |
| [Jenkins](https://insper.github.io/platform/2026.2/exercises/jenkins/){target="_blank"} | substituído por GitHub Actions (build, testes, Trivy, push no ECR e deploy no EKS) | Leonardo (pipeline) e **eu** (AWS e deploy no EKS) |
| [MiniKube](https://insper.github.io/platform/2026.2/exercises/minikube/){target="_blank"} | `k8s/k8s.yaml` de cada serviço (ConfigMap, Deployment, Service; Secrets criados fora do Git) rodando no mesmo cluster: Kubernetes do Docker Desktop em 08/11 e EKS em 08/11 e 16/11 | **eu** (account, auth, notification) e Leonardo (gateway) |
| [Bottlenecks](https://insper.github.io/platform/2026.2/exercises/bottlenecks/){target="_blank"} | 7 no grupo, 2 meus | [Meus bottlenecks](bottlenecks.md) |
| [Projeto: AWS](https://insper.github.io/platform/2026.2/exercises/project/aws/){target="_blank"} | feito | **eu**: [AWS](projeto/aws.md) |
| [Projeto: EKS](https://insper.github.io/platform/2026.2/exercises/project/eks/){target="_blank"} | feito; o teste de carga do grupo com HPA rodou no `eks-store` em 19/11 e a demonstração do vídeo em 20/11 | **eu**: [EKS](projeto/eks.md) |
| [Projeto: CI/CD](https://insper.github.io/platform/2026.2/exercises/project/ci-cd/){target="_blank"} | feito | Leonardo (eu: OIDC e o deploy dos meus serviços) |
| [Projeto: Load Testing](https://insper.github.io/platform/2026.2/exercises/project/load-testing/){target="_blank"} e [Costs & PaaS](https://insper.github.io/platform/2026.2/exercises/project/costs/){target="_blank"} | feitos no compose, num cluster kind e no EKS (19/11, HPA de 2 para 6 réplicas): [resultado](https://pna26.github.io/platform/pt/devops/load-testing/){target="_blank"} | Gabriel |

## Código-fonte

| Atividade | Código |
|---|---|
| Account (Hands-on 1) | [pna26/account](https://github.com/pna26/account){target="_blank"} · [pna26/account-service](https://github.com/pna26/account-service){target="_blank"} |
| Auth (Hands-on 3) | [pna26/auth](https://github.com/pna26/auth){target="_blank"} · [pna26/auth-service](https://github.com/pna26/auth-service){target="_blank"} |
| Kubernetes (Hands-on 5 / MiniKube) | `k8s/k8s.yaml` em cada repositório de serviço; cluster em [`setup/eks-store.yaml`](https://github.com/pna26/platform/blob/main/setup/eks-store.yaml){target="_blank"} |
| AWS | [`setup/aws/`](https://github.com/pna26/platform/tree/main/setup/aws){target="_blank"} (políticas da role do CI) |
| Mensageria (Hands-on 10) | [pna26/notification-service](https://github.com/pna26/notification-service){target="_blank"} · outbox em `account-service/src/main/java/store/account/outbox/` · `compose.yaml`, `k8s/kafka.yaml` e `scripts/messaging-test.sh` no [platform](https://github.com/pna26/platform){target="_blank"} |
| RBAC | `AccessPolicy` no [gateway-service](https://github.com/pna26/gateway-service){target="_blank"}, claim `role` no auth-service, coluna `role` no account-service |
