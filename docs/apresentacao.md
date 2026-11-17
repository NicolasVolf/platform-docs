# Apresentação

A apresentação desta edição é um **vídeo de 1 a 2 minutos** do grupo, com o projeto e os bottlenecks.
Roteiro e slides completos: [Apresentação do grupo](https://pna26.github.io/platform/apresentacao/){target="_blank"}.

## Tópicos

| Tópico pedido pelo curso | No vídeo |
|---|---|
| Introdução ao projeto | loja em microsserviços: cadastro, login e câmbio, tudo entrando por um gateway |
| Arquitetura | Nginx → réplicas do gateway → Auth, Account, Exchange; Redis, PostgreSQL/RDS e Kafka; tudo no EKS, com deploy pelo GitHub Actions ([Projeto](projeto/index.md)) |
| Demonstração | no EKS (a minha parte, abaixo) |
| Desafios: bottlenecks | cada integrante os seus, com o número que mostra o efeito |
| Conclusão e próximos passos | custo de ~US$ 0,27/h com tudo ligado; HTTPS, Kafka com disco persistente e Multi-AZ no banco |

## A minha parte

| Trecho | Tela | O que mostrar |
|---|---|---|
| Demonstração no EKS | terminal | `kubectl get pods -n store`; cadastro e login pelo ELB; `GET /accounts` sem token (`401`) |
| Mensageria | Mailpit / logs do notification | o e-mail de boas-vindas chegando pelo Kafka; o cadastro continua em ~15 ms com o Kafka ou o consumidor fora |
| RBAC | terminal | um `USER` tentando listar e apagar contas alheias → `403`; a própria conta → `200` |

## Próximos passos (das minhas partes)

- HTTPS com domínio próprio e certificado no load balancer, para o navegador aceitar o cookie `Secure` no EKS.
- Kafka com disco persistente e 3 brokers (ou Amazon MSK) e o lag do consumidor no Grafana.
- `409` para e-mail duplicado e hash de senha com BCrypt.
- Tokens de vida curta com *refresh token*, para rebaixar um `ADMIN` na hora.

## Vídeo

!!! warning "Em gravação"

    O vídeo é gravado com a plataforma inteira no EKS e entra aqui e na [página inicial](index.md)
    assim que estiver pronto.

<!-- vídeo: <iframe width="100%" height="470" src="https://www.youtube.com/embed/<id>" allowfullscreen></iframe> -->
