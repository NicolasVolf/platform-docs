# Apresentação

A apresentação desta edição é um **vídeo de 1 a 2 minutos** do grupo, com o projeto e os bottlenecks.
Roteiro completo: [Apresentação do grupo](https://pna26.github.io/platform/pt/apresentacao/){target="_blank"}.

## Vídeo

<video controls preload="metadata" width="100%" src="../assets/apresentacao-pna26.mp4"></video>

1 min 45 s, sem narração: cada cena tem uma legenda curta explicando o que aparece na tela. A
demonstração foi gravada ao vivo em 20/11/2026 pelo ELB do `eks-store`, com o ambiente que eu subi e o
deploy feito pelo CI; o teste com HPA é a gravação de 19/11 no mesmo cluster.

## Tópicos

| Tópico pedido pelo curso | No vídeo |
|---|---|
| Introdução ao projeto | loja em microsserviços: cadastro, login e câmbio, tudo entrando por um gateway |
| Arquitetura | Nginx → réplicas do gateway → Auth, Account, Exchange; Redis, PostgreSQL/RDS e Kafka; tudo no EKS, com deploy pelo GitHub Actions ([Projeto](projeto/index.md)) |
| Demonstração | ao vivo no EKS, pelo ELB público |
| Desafios: bottlenecks | uma cena por bottleneck, com o número que mostra o efeito |
| Conclusão e próximos passos | custo de ~US$ 0,29/h com tudo ligado; HTTPS, Kafka com disco persistente e Multi-AZ no banco |

## A minha parte no vídeo

| Cena | O que aparece | Onde está |
|---|---|---|
| Plataforma no EKS (0:21–0:50) | cluster, RDS e deploy pelo CI que eu montei; pelo ELB: cadastro e login no Account/Auth (cookie `HttpOnly`), `401` sem token e com `id-account` forjado | [EKS](projeto/eks.md), [Account](projeto/account.md), [Auth](projeto/auth.md) |
| RBAC (na demonstração) | uma conta `USER` recebe `200` na própria conta e `403` ao listar as contas dos outros | [RBAC](bottlenecks.md#autorizacao-rbac) |
| Bottleneck: mensageria | cadastro em ~15 ms com o consumidor **ou** o Kafka fora; 0 eventos perdidos, 0 e-mails duplicados | [Mensageria](bottlenecks.md#mensageria-kafka) |

## Próximos passos (das minhas partes)

- HTTPS com domínio próprio e certificado no load balancer, para o navegador aceitar o cookie `Secure` no EKS.
- Kafka com disco persistente e 3 brokers (ou Amazon MSK) e o lag do consumidor no Grafana.
- `409` para e-mail duplicado e hash de senha com BCrypt.
- Tokens de vida curta com *refresh token*, para rebaixar um `ADMIN` na hora.
