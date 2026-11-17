# EKS

!!! info "Página completa do grupo, com todos os comandos: [EKS](https://pna26.github.io/platform/devops/eks/){target='_blank'}"

A plataforma roda num cluster **Amazon EKS** (`eks-store`, região `us-east-2`), com o banco do Account
num **Amazon RDS for PostgreSQL**. Tudo fica no namespace `store`. O cluster é criado pelo `eksctl` a
partir de um arquivo versionado e **apagado quando não está em uso**, porque é o item mais caro do projeto.

``` mermaid
flowchart LR
    user([internet]) --> elb[ELB da AWS<br/>Service lb · LoadBalancer]
    subgraph vpc[VPC 10.20.0.0/16 · us-east-2a e 2b]
        subgraph pub[sub-redes públicas]
            elb
            nat[NAT gateway]
        end
        subgraph priv[sub-redes privadas · 2 nós t3.medium]
            lb[lb · Nginx] --> gw[gateway]
            gw --> auth & account & exchange
            auth --> account
            account -.-> kafka[[Kafka]] -.-> notification
        end
        rds[(RDS PostgreSQL 17<br/>db.t4g.micro)]
    end
    elb --> lb
    account -->|5432| rds
    notification -->|5432| rds
    exchange --> nat
    priv -.->|pull| ecr[(ECR)]
```

## Decisões

| Peça | Escolha | Por quê |
|---|---|---|
| Kubernetes | 1.37 | versão padrão do EKS, em suporte padrão (as de suporte estendido custam 6× mais por hora) |
| Nós | 2× `t3.medium`, managed node group | os *requests* de memória somam ~2,6 GiB; `t3.small` não deixava folga para *rolling update* nem para o teste de carga |
| Rede | VPC do eksctl, 2 zonas, nós **só em sub-redes privadas** | os nós não ficam expostos; só o ELB recebe tráfego da internet |
| Saída para a internet | 1 NAT gateway | os nós baixam imagens e o exchange chama a API de câmbio; um NAT por zona dobraria o custo |
| Acesso ao cluster | `authenticationMode: API` (access entries) | permissões do Kubernetes dadas a roles da AWS sem editar o `aws-auth` |
| Imagens | ECR | os pacotes do GHCR são privados; os nós já têm leitura no ECR |
| Banco | RDS PostgreSQL 17, `db.t4g.micro`, 20 GB gp3, sem acesso público | a página do EKS do curso pede RDS ou DynamoDB; o *security group* aceita a porta 5432 **só dos nós** |

``` { .yaml .copy linenums="1" title="setup/eks-store.yaml (pna26/platform)" }
--8<-- "https://raw.githubusercontent.com/pna26/platform/e7730f9ea57355a35e83d6a8e271f47326c4fe6a/setup/eks-store.yaml"
```

## Manifestos dos meus serviços

Cada serviço tem o próprio `k8s/k8s.yaml` (ConfigMap, Deployment e Service), como no exercício MiniKube
e no Hands-on 5. Padrões que segui em todos:

- Service `ClusterIP` com o **mesmo nome do compose** (`account`, `auth`): os Feign clients
  (`http://account:8080`) funcionam sem mudança;
- container sem root (uid 1001), sem escalada de privilégio, `/tmp` num `emptyDir`;
- *requests*/*limits* de memória e probes TCP na 8080;
- **nenhum Secret no Git:** o pipeline faz `kubectl apply -f k8s/` a cada deploy, e um Secret de
  exemplo ali sobrescreveria o real. `account-secrets`, `auth-secrets` e `notification-secrets` são
  criados uma vez por cluster, com a senha aleatória do RDS lida de um arquivo temporário (nunca na tela).

``` { .yaml .copy linenums="1" title="k8s/k8s.yaml (pna26/account-service)" }
--8<-- "https://raw.githubusercontent.com/pna26/account-service/a66e0b39c87865509c30f195f536b78365831186/k8s/k8s.yaml"
```

## Permissões do CI no cluster

O pipeline faz o deploy sozinho em todo push na `main` quando a variável da org `EKS_CLUSTER` existe. A
role OIDC do CI ([AWS](aws.md)) ganhou uma **access entry** com `AmazonEKSAdminPolicy` **só no namespace
`store`**.

No deploy de 16/11 apareceu um problema que a documentação da AWS não deixa claro: ao criar um `Role`
(o do Prometheus), o Kubernetes confere se quem cria já tem todas as permissões que está dando (proteção
contra escalada de privilégio). Essa conferência só olha o RBAC do próprio Kubernetes e **não enxerga a
access policy da AWS**. A correção foi colocar a role num grupo `store-deployers` e dar a esse grupo o
papel `admin` com um `RoleBinding`, **só no `store`**:

``` bash
aws eks update-access-entry --cluster-name eks-store --region us-east-2 \
  --principal-arn arn:aws:iam::637423273088:role/github-actions-pna26 --kubernetes-groups store-deployers
kubectl create rolebinding ci-store-admin -n store --clusterrole=admin --group=store-deployers
# conferência: "yes" no store e "no" fora dele
kubectl auth can-i create roles -n store   --as=ci --as-group=store-deployers
kubectl auth can-i create roles -n default --as=ci --as-group=store-deployers
```

## Resultados

| Data | O que foi feito | Resultado |
|---|---|---|
| 08/11/2026 | primeira subida, deploy **manual** dos 4 serviços + infra | 11 pods prontos; o Flyway criou o schema no RDS; **smoke test passou** pelo ELB (cadastro, login, `401` sem token e com token forjado, 2 réplicas do gateway respondendo). Ambiente apagado no mesmo dia |
| 16/11/2026 | segunda subida, deploy **pelo CI** | os pipelines de account-service, auth-service e notification-service fizeram build, testes, Trivy, push no ECR, `kubectl apply` e `rollout status` sozinhos, todos verdes. Register `200`, login `200`, senha errada `401`; o evento chegou ao notification. Ambiente apagado e a conferência de recursos saiu vazia |

Runs do deploy pelo CI (16/11):
[account-service](https://github.com/pna26/account-service/actions/runs/43856117521){target="_blank"} ·
[auth-service](https://github.com/pna26/auth-service/actions/runs/43856123470){target="_blank"} ·
[notification-service](https://github.com/pna26/notification-service/actions/runs/43856129429){target="_blank"}

## Custo

| Item | US$/h |
|---|---:|
| Control plane do EKS | 0,100 |
| 2× EC2 `t3.medium` | 0,083 |
| NAT gateway | 0,045 |
| Load balancer | 0,025 |
| RDS `db.t4g.micro` + 20 GB | 0,019 |
| **Total** | **~0,27** |

Com o budget de US$ 20/mês, são cerca de 70 horas ligado. Por isso o ambiente é criado para testar e
apagado em seguida, numa ordem certa: o ELB e o RDS ficam dentro da VPC do eksctl e impedem que ela seja
apagada se ficarem para trás.

## Problemas encontrados

| Problema | Solução |
|---|---|
| `docker login` no ECR falha no Windows (token grande demais para o Gerenciador de Credenciais) | `docker-credential-ecr-login` em `credHelpers` |
| `infra` falha ao criar o `Role` do Prometheus (anti-escalada do Kubernetes) | grupo `store-deployers` + `RoleBinding` para `admin` no `store` |
| exchange devolve `503` no EKS | a API de câmbio gratuita responde `429` para IPs da AWS; o exchange do Gabriel trata com cache e `503` |
| `gh variable set --org` devolve `403` | escopo `admin:org` no `gh` só enquanto precisa, removido depois |
