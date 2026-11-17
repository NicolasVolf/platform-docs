# Uso de IA

A disciplina permite usar IA desde que o uso seja documentado e que cada integrante saiba explicar e
defender todas as partes. O registro completo do grupo, por integrante, está em
[Uso de IA](https://pna26.github.io/platform/uso-de-ia/){target="_blank"}; aqui está o resumo da minha parte.

**Ferramenta:** Claude Code (Anthropic), no terminal e no app desktop, com acesso ao clone local do
`platform`, ao GitHub da organização e, nas etapas da AWS, ao AWS CLI logado por SSO.

**Como usei:** para **consultar o que deveria ser feito corretamente** e para **estruturar o código**.
Antes de cada tarefa, a IA lia o material do curso (Hands-on 1, 3 e de mensageria, e as páginas de EKS e
de bottlenecks) e explicava o que o professor espera. Depois montava a estrutura (classes, migrations,
manifestos, comandos) seguindo os padrões do curso e do grupo. **Eu revisava cada mudança**, pedia
explicação do que não estava claro e só seguia quando entendia o que tinha sido feito e por quê.

| Parte | Com IA? | Como foi verificado |
|---|---|---|
| [Account](projeto/account.md) | Sim, estrutura a partir do Hands-on 1 | todas as rotas testadas no compose; smoke test no CI |
| [Auth](projeto/auth.md) | Sim, estrutura a partir do Hands-on 3 | register, login, token adulterado (`401`), `me` e logout; `JwtServiceTest` |
| [AWS](projeto/aws.md) | Sim, explicação de cada passo; **os comandos e os cliques no console fui eu que fiz** | a IA explicava antes e conferia o resultado depois |
| [EKS](projeto/eks.md) | Sim: manifestos, `eks-store.yaml`, RDS e a ordem de criar e apagar | deploy manual e smoke test em 08/11; deploy pelo CI em 16/11; conferência de que nada ficou cobrando |
| [Mensageria](bottlenecks.md#mensageria-kafka) | Sim: código, compose, manifestos, script de teste e a página | `scripts/messaging-test.sh` (5 experimentos) no compose; evento no EKS em 16/11 |
| [RBAC](bottlenecks.md#autorizacao-rbac) | Sim: a IA achou a falha lendo o gateway e o account, e estruturou a solução | 10 testes manuais no compose e 13 automatizados no CI |
| Esta documentação | Sim, a partir do template do professor e das páginas do grupo | `mkdocs build --strict` |

**O que eu fiz sem IA:**

- **AWS na mão:** rodei os comandos e fiz os cliques no console (budget, SSO, ECR, role do CI).
- **Decisões de projeto:** região `us-east-2` e alerta de US$ 20/mês; RDS PostgreSQL no EKS; nós só em
  sub-redes privadas e RDS sem acesso público; apagar o ambiente sempre que não está em uso; na
  mensageria, o e-mail de boas-vindas como caso de uso, o consumidor num serviço separado e Kafka (o
  material do professor) em vez do RabbitMQ do plano inicial; no RBAC, conta nova sempre `USER` e
  `ADMIN` só pelo banco.
- **Revisão e testes:** li e entendi cada mudança antes de ir para o GitHub, e testei no compose e no cluster.
- **Regras para a IA:** nenhum segredo no Git, na tela ou em logs; commits sem coautoria da IA; mudanças
  no repositório de outro integrante só por PR; aprovação minha em cada ação externa.

**Onde a IA errou e foi corrigido:**

- **OIDC recusado no primeiro CI:** a condição da role não casava com o *subject* imutável do GitHub.
- **Nomes do ECR** diferentes dos que o pipeline usava: recriados.
- **Permissão do CI no cluster:** nem a `AmazonEKSEditPolicy` nem a `AmazonEKSAdminPolicy` deixavam criar o
  `Role` do Prometheus; a correção foi um `RoleBinding` nativo, só no `store`.
- **Chave reserva do JWT** no `application.yaml`: removida na revisão de segurança; agora o Auth não sobe
  sem `JWT_SECRET_KEY`.
