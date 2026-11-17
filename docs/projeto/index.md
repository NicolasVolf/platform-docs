# Projeto

Uma plataforma de microsserviços para uma loja: o usuário se cadastra, faz login e consulta câmbio
entre moedas. Todas as requisições entram por um único ponto (o **gateway**, atrás de um balanceador
**Nginx**), que valida o token JWT e repassa a identidade do usuário para os serviços. Roda com Docker
Compose na máquina e no **EKS** (Kubernetes da AWS), com deploy pelo **GitHub Actions**.

``` mermaid
flowchart LR
    internet([Internet]) ==> lb["lb · Nginx"]
    subgraph trusted [Camada confiável]
        lb ==> gw["gateway<br/>(réplicas)"]
        gw --> auth:::meu
        gw --> account:::meu
        gw --> exchange
        gw <-->|token → id + papel| redis[(Redis)]
        auth --> account
        account --> db[(PostgreSQL / RDS)]
        account -.->|evento| kafka[[Kafka]]:::meu -.-> notification:::meu
    end
    exchange --> api3([API de câmbio])
    classDef meu fill:#1f5f99,stroke:#2a6fb0,color:#fff
```

Em azul, o que eu implementei. A AWS (conta, ECR, role do CI) e o cluster EKS onde tudo isso roda
também são meus.

| Integrante | Responsabilidades |
|---|---|
| Leonardo | Gateway, CI/CD, balanceamento de carga, cache (Redis), varredura de vulnerabilidades, documentação (MkDocs) |
| **Nicolas** (eu) | Auth, Account, AWS, EKS, mensageria (Kafka), autorização (RBAC) |
| Gabriel | Exchange, testes de carga, observabilidade, cache da cotação, PaaS/custos/SLA |

Arquitetura completa e contrato da API: [Arquitetura](https://pna26.github.io/platform/arquitetura/){target="_blank"}.

## Minhas partes

| Página | Resumo |
|---|---|
| [Account](account.md) | dono das contas: CRUD, hash da senha, login para o Auth, outbox |
| [Auth](auth.md) | cadastro e login, JWT assinado no cookie, `/auth/solve` para o gateway |
| [AWS](aws.md) | conta, região, budget, SSO, ECR e a role OIDC do GitHub Actions |
| [EKS](eks.md) | cluster `eks-store`, RDS, Secrets, permissões do CI e o ciclo criar/testar/apagar |

## Código-fonte

Todo o código está nos repositórios da organização [pna26](https://github.com/pna26){target="_blank"}. O
repositório principal, [pna26/platform](https://github.com/pna26/platform){target="_blank"}, junta os
serviços como **submódulos** em `api/`:

``` bash
git clone --recurse-submodules https://github.com/pna26/platform.git
cd platform
./scripts/build.sh
docker compose up -d --build
```

Os trechos de código destas páginas vêm direto dos repositórios, fixados no commit da `main` de 17/11/2026.

## Status (17/11/2026)

| Item | Situação |
|---|---|
| Account, Auth, notification-service | prontos; rodando no compose, no CI e no EKS |
| AWS | pronta; o CI entra na AWS por OIDC desde 10/11 |
| EKS | testado com deploy manual em 08/11; deploy automático pelo CI comprovado em 16/11 (account, auth e notification). Cluster apagado fora dos testes, por custo |
| Mensageria | pronta no compose (5 experimentos) e no EKS (evento chegou ao notification em 16/11) |
| RBAC | pronto no compose e no CI (13 testes novos) |
| Falta | uma segunda subida do cluster para o gateway, o exchange e o teste de carga com HPA do grupo |
