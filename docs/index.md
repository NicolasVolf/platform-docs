# Nicolas · Platform 2026.2

???+ info inline end "Disciplina"

    Plataformas, Microsserviços, DevOps e APIs
    Insper · 2026.2

**Aluno:** Nicolas Volf Martins da Silva ([@NicolasVolf](https://github.com/NicolasVolf){target="_blank"})

**Grupo:** pna26

1. Leonardo de Souza Lima e Silva ([@souzxxx](https://github.com/souzxxx){target="_blank"})
1. Nicolas Volf Martins da Silva (eu)
1. Gabriel ([@gabrielalmeidaconde](https://github.com/gabrielalmeidaconde){target="_blank"})

Esta é a minha documentação individual. O projeto foi feito em grupo; aqui estão as partes que eu
implementei, os meus dois bottlenecks, os hands-on e exercícios, e os links de tudo o que foi usado.

**Documentação completa do grupo:** [pna26.github.io/platform](https://pna26.github.io/platform/){target="_blank"}

## Vídeo de apresentação (1–2 min)

!!! warning "Em gravação"

    O vídeo do grupo é gravado com a plataforma no EKS (roteiro em [Apresentação](apresentacao.md)) e
    entra aqui assim que estiver pronto.

<!-- vídeo: <iframe width="100%" height="470" src="https://www.youtube.com/embed/<id>" allowfullscreen></iframe> -->

## Entregas

- [x] Account (5%): [Projeto → Account](projeto/account.md)
- [x] Auth (5%): [Projeto → Auth](projeto/auth.md)
- [x] AWS setup (5%): [Projeto → AWS](projeto/aws.md)
- [x] EKS (10%): cluster `eks-store`, RDS e deploy automático pelo CI comprovado em 16/11/2026, em [Projeto → EKS](projeto/eks.md)
- [x] Bottleneck 1: mensageria com Kafka, em [Meus bottlenecks](bottlenecks.md#mensageria-kafka)
- [x] Bottleneck 2: autorização com RBAC, em [Meus bottlenecks](bottlenecks.md#autorizacao-rbac)
- [x] Hands-on e exercícios: [Hands-on e exercícios](atividades.md)
- [x] Uso de IA documentado: [Uso de IA](uso-de-ia.md)
- [ ] Vídeo do grupo (1–2 min)

## O que eu fiz

| Entrega | O quê | Detalhes |
|---|---|---|
| Microsserviço **Account** | Cadastro de contas em PostgreSQL + Flyway, senha só como hash, rota de login usada pelo Auth | [Account](projeto/account.md) |
| Microsserviço **Auth** | Cadastro, login, JWT no cookie `__store_jwt_token` e validação do token para o gateway | [Auth](projeto/auth.md) |
| **AWS** | Região, alerta de custo, SSO no AWS CLI, ECR e a role OIDC do GitHub Actions (sem chave de acesso) | [AWS](projeto/aws.md) |
| **EKS** | Cluster `eks-store`, RDS PostgreSQL, Secrets, permissões do CI no cluster e o roteiro de criar e apagar | [EKS](projeto/eks.md) |
| Bottleneck: **mensageria** | Kafka com *transactional outbox* e o `notification-service`, um consumidor idempotente com DLT | [Mensageria](bottlenecks.md#mensageria-kafka) |
| Bottleneck: **autorização (RBAC)** | Papéis `USER`/`ADMIN`, claim `role` no JWT e regras no gateway | [RBAC](bottlenecks.md#autorizacao-rbac) |

## Repositórios usados

| Repositório | O que é | Responsável |
|---|---|---|
| [pna26/account](https://github.com/pna26/account){target="_blank"} | Interface (Feign) e DTOs do Account | **eu** |
| [pna26/account-service](https://github.com/pna26/account-service){target="_blank"} | Implementação do Account (PostgreSQL + Flyway + outbox) | **eu** |
| [pna26/auth](https://github.com/pna26/auth){target="_blank"} | Interface (Feign) e DTOs do Auth | **eu** |
| [pna26/auth-service](https://github.com/pna26/auth-service){target="_blank"} | Implementação do Auth (JWT) | **eu** |
| [pna26/notification-service](https://github.com/pna26/notification-service){target="_blank"} | Consumidor Kafka: e-mail de boas-vindas | **eu** |
| [pna26/platform](https://github.com/pna26/platform){target="_blank"} | Repositório principal: documentação, `compose.yaml`, Nginx, manifestos k8s compartilhados, pipeline reutilizável e os serviços como submódulos | Leonardo (eu: `setup/`, Kafka, páginas de AWS/EKS/mensageria/RBAC) |
| [pna26/gateway-service](https://github.com/pna26/gateway-service){target="_blank"} | API gateway (Spring Cloud Gateway) | Leonardo (eu: regras do RBAC, por PR) |
| [pna26/exchange-service](https://github.com/pna26/exchange-service){target="_blank"} | Câmbio (Python/FastAPI) | Gabriel |
| [NicolasVolf/platform-docs](https://github.com/NicolasVolf/platform-docs){target="_blank"} | Esta documentação individual | **eu** |

Quadro de tarefas do grupo: [Projeto Platform 2026.2](https://github.com/orgs/pna26/projects/1){target="_blank"}.
