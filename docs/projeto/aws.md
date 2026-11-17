# AWS

!!! info "Página completa do grupo: [AWS](https://pna26.github.io/platform/devops/aws/){target='_blank'}"

Montei a **base da conta AWS** do grupo: região, alerta de custo, acesso pela linha de comando, os
repositórios de imagem (ECR) e a role que o GitHub Actions usa para entrar na AWS **sem chave de
acesso**. Os comandos e os cliques no console fui eu que fiz, um passo de cada vez (ver
[Uso de IA](../uso-de-ia.md)).

``` mermaid
flowchart LR
    subgraph gh[GitHub · org pna26]
        ci[GitHub Actions<br/>service.yaml]
    end
    subgraph aws[Conta do curso · us-east-2]
        oidc[provedor OIDC<br/>token.actions.githubusercontent.com]
        role[role IAM<br/>github-actions-pna26]
        ecr[(ECR<br/>5 repositórios)]
        eks[EKS eks-store]
        budget[budget pna26-mensal<br/>US$ 20/mês]
    end
    ci -->|"1. token OIDC assinado pelo GitHub"| oidc
    oidc -->|"2. confere aud e sub"| role
    role -->|"3. credenciais temporárias (1 h)"| ci
    ci -->|"push da imagem"| ecr
    ci -->|"kubectl apply (access entry)"| eks
    eks -.->|"pull"| ecr
```

## O que foi feito

| Passo | O que eu fiz | Por quê |
|---|---|---|
| 1. Alerta de custo | budget `pna26-mensal`, US$ 20/mês, e-mail em 85 % e 100 % do real e 100 % do previsto | feito primeiro, antes de criar qualquer recurso pago. Só avisa, não bloqueia |
| 2. Região | **`us-east-2` (Ohio)** para tudo | tem EKS, RDS e ECR com os preços de `us-east-1`; uma região só evita recurso esquecido em outro lugar |
| 3. Acesso pela linha de comando | `aws configure sso` (perfil `default`), **sem *access key*** | o roteiro do curso sugere uma *access key*; o SSO gera credenciais temporárias, que expiram sozinhas. Uma chave fixa vale até ser revogada e, se vazar, dá acesso à conta |
| 4. ECR | 5 repositórios `*-service` (gateway, auth, account, exchange, notification), com *scan on push* | os nomes batem com os que o pipeline já usava, então nada mudou no CI |
| 5. CI sem chave | provedor OIDC do GitHub + role `github-actions-pna26` | o mesmo princípio do passo 3, agora para as máquinas |

## Role do GitHub Actions (OIDC)

1. O job pede ao GitHub um **token OIDC** (por isso o `ci.yaml` de cada serviço tem `id-token: write`); o
   token é assinado pelo GitHub e diz de qual repositório e branch ele veio.
2. A action `aws-actions/configure-aws-credentials` troca esse token por **credenciais temporárias** da
   role (`sts:AssumeRoleWithWebIdentity`).
3. A AWS só aceita se as **condições** da role baterem; as credenciais expiram em 1 hora.

=== "Confiança (quem pode assumir)"

    ``` { .json .copy linenums="1" title="setup/aws/github-actions-trust.json" }
    --8<-- "https://raw.githubusercontent.com/pna26/platform/e7730f9ea57355a35e83d6a8e271f47326c4fe6a/setup/aws/github-actions-trust.json"
    ```

=== "Permissões (o que pode fazer)"

    ``` { .json .copy linenums="1" title="setup/aws/github-actions-permissions.json" }
    --8<-- "https://raw.githubusercontent.com/pna26/platform/e7730f9ea57355a35e83d6a8e271f47326c4fe6a/setup/aws/github-actions-permissions.json"
    ```

As permissões seguem o **menor privilégio**: a role não cria nem apaga recursos da AWS, só envia imagens
aos 5 repositórios do ECR e lê o cluster (`eks:DescribeCluster`). Dentro do Kubernetes, ela só mexe no
namespace `store` (ver [EKS](eks.md#permissoes-do-ci-no-cluster)).

## Problemas encontrados

| Problema | Causa | Solução |
|---|---|---|
| Primeiro CI com OIDC falhou: `Not authorized to perform sts:AssumeRoleWithWebIdentity` (10/11) | os repositórios da org usam o **subject imutável** do GitHub (`repo:pna26@<id da org>/<repo>@<id>:ref:...`), e a condição da role era `repo:pna26/*` | condição trocada para `repo:pna26@317011053/*`. Fica até mais seguro: o id numérico não muda, então outra org que um dia use o nome `pna26` não casa com a regra |
| Repositórios do ECR com o nome errado | os primeiros (`gateway`, `auth`...) não batiam com os nomes do pipeline (`*-service`) | recriados com o nome certo; os antigos, vazios, apagados |
| `docker login` no ECR falha no Windows | o token do ECR com SSO tem ~3 KB e o Gerenciador de Credenciais do Windows aceita até 2560 bytes | `docker-credential-ecr-login` (vem com o Docker Desktop) em `credHelpers` |

## Melhorias possíveis

| Melhoria | Efeito |
|---|---|
| Restringir a confiança a `...:ref:refs/heads/main` | só pushes na `main` assumiriam a role |
| *Secret scanning* e *push protection* na org | o GitHub bloqueia um push com uma chave da AWS por engano |
| Uma role por repositório | o CI de um serviço não conseguiria enviar imagem para o repositório de outro |
