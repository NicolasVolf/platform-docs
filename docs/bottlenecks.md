# Meus bottlenecks

O curso pede **ao menos 2 bottlenecks por integrante**. Implementei dois, ambos da
[tabela de bottlenecks do curso](https://insper.github.io/platform/2026.2/exercises/bottlenecks/){target="_blank"}.
Cada um tem a página completa (problema, solução, código, evidência) na documentação do grupo; aqui
está o resumo com os números que mostram o efeito.

| Bottleneck | Número que mostra o efeito | Página completa |
|---|---|---|
| Mensageria (Kafka) | cadastro em **~15 ms** mesmo com o consumidor **ou** o Kafka fora do ar; **0** eventos perdidos e **0** e-mails duplicados | [Mensageria](https://pna26.github.io/platform/bottlenecks/messaging/){target="_blank"} |
| Autorização (RBAC) | as 4 brechas (listar, ver, apagar contas alheias e usar a rota interna) passaram a dar **403**; 10/10 testes manuais e 13 testes automatizados | [Autorização (RBAC)](https://pna26.github.io/platform/bottlenecks/rbac/){target="_blank"} |

## Mensageria (Kafka)

**Problema:** depois do cadastro, a loja manda um **e-mail de boas-vindas**. Se o `account` chamasse o
serviço de e-mail dentro do `POST /accounts`, o cadastro ficaria **acoplado** a ele: o usuário esperaria
a latência do provedor, uma queda do e-mail viraria uma queda do cadastro, e um pico de cadastros bateria
no provedor no mesmo ritmo.

**Solução:** segue o Hands-on de mensageria do curso (Kafka, *transactional outbox*, consumidor
idempotente e DLT), adaptado para o cadastro de contas.

``` mermaid
flowchart LR
    u([Usuário]) -->|"POST /auth/register"| auth --> account
    subgraph tx ["uma transação no PostgreSQL"]
        account -->|"INSERT"| acc[("accounts")]
        account -->|"INSERT AccountCreated"| ob[("outbox")]
    end
    relay["OutboxRelay<br/>(a cada 200 ms)"] -->|"lê pendentes"| ob
    relay -->|"chave = id da conta<br/>acks=all"| k[["Kafka<br/>account-events (6 partições)"]]
    k --> n["notification-service"]
    n -->|"dedupe por eventId"| pe[("processed_events")]
    n -->|SMTP| mail(["Mailpit"])
    n -. "falha permanente" .-> dlt[["account-events-dlt"]]
```

| Decisão | Motivo |
|---|---|
| **Transactional outbox** | gravar a conta e publicar no Kafka são duas escritas em sistemas diferentes; se o processo cair entre elas, o evento se perde. Com o outbox, conta e evento entram **na mesma transação** e um relay publica depois do commit |
| Relay com `FOR UPDATE SKIP LOCKED` | várias réplicas do `account` podem rodar o relay sem publicar a mesma linha duas vezes |
| Produtor com `acks=all` e idempotência | o broker só confirma depois de gravar, e um reenvio não duplica a mensagem |
| Chave = **id da conta**, 6 partições | eventos da mesma conta ficam em ordem (mesma partição); 6 é o teto de consumidores em paralelo |
| Consumidor **idempotente** | a entrega é *at-least-once*: o `eventId` é gravado em `processed_events` na mesma transação do e-mail; se já existe, é ignorado |
| 3 retries com *backoff* e **DLT** | falha temporária é tentada de novo; depois disso a mensagem vai para a DLT e a partição continua andando. Mensagem inválida vai direto para a DLT |
| O evento não leva senha nem hash | só o mínimo para o e-mail |

=== "OutboxRelay.java (account-service)"

    ``` { .java .copy linenums="1" }
    --8<-- "https://raw.githubusercontent.com/pna26/account-service/a66e0b39c87865509c30f195f536b78365831186/src/main/java/store/account/outbox/OutboxRelay.java"
    ```

=== "NotificationService.java (notification-service)"

    ``` { .java .copy linenums="1" }
    --8<-- "https://raw.githubusercontent.com/pna26/notification-service/b3dc4c68118bc24cec4cae97ed3845ab6c444a65/src/main/java/store/notification/NotificationService.java"
    ```

=== "KafkaConfig.java (retries e DLT)"

    ``` { .java .copy linenums="1" }
    --8<-- "https://raw.githubusercontent.com/pna26/notification-service/b3dc4c68118bc24cec4cae97ed3845ab6c444a65/src/main/java/store/notification/KafkaConfig.java"
    ```

**Resultados** (`scripts/messaging-test.sh` no compose, 08/11/2026; 20 cadastros por experimento e
latência simulada do e-mail de 50 ms):

| Experimento | O que aconteceu |
|---|---|
| 1. Caminho feliz | 20 cadastros, **média de 16 ms** por cadastro; 20 e-mails no Mailpit |
| 2. `notification` parado | cadastro continuou em **15 ms**; o lag subiu para 20 mensagens guardadas no Kafka; ao religar, drenou em 9 s e chegaram os 20 e-mails |
| 3. Kafka parado | cadastro continuou em **15 ms**; os 20 eventos ficaram no outbox e foram publicados quando o Kafka voltou: **nenhum perdido** |
| 4. Reprocessamento (offsets voltados ao início) | o tópico inteiro foi relido e **nenhum e-mail novo** foi enviado |
| 5. Mensagem envenenada | foi para a DLT sem retry; os cadastros seguintes geraram os e-mails normalmente |

No EKS, em 16/11/2026, o `AccountCreated` de um cadastro feito no cluster chegou ao `notification-service`.

**Limites:** um broker só e sem disco persistente no EKS (`emptyDir`); em produção, 3 brokers com
replicação ou o Amazon MSK. O lag do consumidor ainda não está no Grafana.

## Autorização (RBAC)

**Problema:** o gateway já garantia **quem** é o usuário, mas não **o que** ele podia fazer. Achei a
falha lendo o gateway e o account: qualquer pessoa logada, com o próprio token, conseguia pelo gateway
**listar todas as contas**, **ver** e **apagar** a conta de outra pessoa e usar a rota interna
`POST /accounts/login`. É o gargalo *Authentication & Authorization* da tabela do curso, e a mitigação
sugerida é RBAC.

**Solução:** dois papéis, `USER` e `ADMIN`. O papel mora no Account, viaja **dentro do JWT assinado** e
é conferido no gateway, o único ponto de entrada:

| Rota escolhida pelo gateway | `USER` | `ADMIN` |
|---|---|---|
| `account` (`/accounts/**`) | só `GET /accounts/{o próprio id}` | tudo |
| `auth`, `exchange` e as demais | tudo (basta estar autenticado) | tudo |

Sem token ou com token inválido continua sendo **401**; token válido sem permissão é **403**.

| Decisão | Motivo |
|---|---|
| Papel **no JWT**, conferido no **gateway** | o gateway já resolve o token em toda requisição (com cache): a checagem não acrescenta chamada de rede |
| A regra olha a **rota escolhida**, não o texto da URL | variações como `/accounts;x=1` ou `/accounts/` caem na mesma regra |
| **Negado por padrão** no `account` | qualquer rota nova do account exige `ADMIN` sem precisar lembrar de atualizar a regra |
| Conta nova é sempre `USER`; `ADMIN` só **direto no banco** | ninguém se autopromove: um `"role": "ADMIN"` no cadastro é ignorado (testado) |
| Token antigo sem o claim `role` vale `USER` | na dúvida, o menor acesso |

O código passou por 4 repositórios: `account` (campo `role` no DTO), `account-service` (coluna `role`,
ver [Account](projeto/account.md#banco-flyway)), `auth-service` (claim `role`, ver
[Auth](projeto/auth.md#jwt)) e `gateway-service` (as regras, por PR para o Leonardo revisar):

``` { .java .copy linenums="1" title="AccessPolicy.java (pna26/gateway-service)" }
--8<-- "https://raw.githubusercontent.com/pna26/gateway-service/5d2144bead5da16c15d27ac42a8227cdc11cfc30/src/main/java/store/gateway/security/AccessPolicy.java"
```

**Resultado** (compose local, 16/11/2026, com duas contas novas, Ana e Bruno):

| Teste | Esperado | Obtido |
|---|---|---|
| Cadastro do Bruno com `"role": "ADMIN"` no corpo | conta `USER` | `USER` |
| Ana (`USER`): `GET /accounts` | 403 | 403 |
| Ana (`USER`): `GET /accounts/{própria}` | 200 | 200 |
| Ana (`USER`): `GET /accounts/{Bruno}` | 403 | 403 |
| Ana (`USER`): `DELETE /accounts/{Bruno}` | 403 | 403 |
| Ana (`USER`): `GET /accounts;x=1` | 403 | 403 |
| Ana (`USER`): `GET /auth/me` | 200 | 200 |
| Ana promovida no banco, ainda com o token antigo: `GET /accounts` | 403 | 403 |
| Ana (`ADMIN`, novo login): `GET /accounts` | 200 | 200 |
| Ana (`ADMIN`): `DELETE /accounts/{Bruno}` | 204 | 204 |

Também no CI: 9 testes novos no gateway e 4 no auth-service (incluindo trocar `USER` por `ADMIN` no
payload do token, que invalida a assinatura → 401), e o `scripts/smoke-test.sh` ganhou as três checagens
de 403.

**Limites:** rebaixar um `ADMIN` só vale quando o token dele expira; só dois papéis fixos; promoção manual
no banco; dentro do cluster os serviços confiam em quem os chama (em produção, *network policies* ou mTLS).
