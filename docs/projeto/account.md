# Account

!!! info "Repositórios: [pna26/account](https://github.com/pna26/account){target='_blank'} (interface) e [pna26/account-service](https://github.com/pna26/account-service){target='_blank'} (implementação)"

    Página completa do grupo: [Account](https://pna26.github.io/platform/servicos/account/){target="_blank"}

O Account é o dono do **cadastro de contas** da loja. Ele:

1. cria, lista, busca e apaga contas (`/accounts`);
2. guarda a senha só como **hash** (a senha nunca é salva nem devolvida);
3. confere e-mail e senha no **login**, a pedido do Auth (`POST /accounts/login`);
4. grava o papel da conta (`USER`/`ADMIN`), usado pelo [RBAC](../bottlenecks.md#autorizacao-rbac);
5. registra um evento `AccountCreated` no **outbox**, na mesma transação da conta, para a
   [mensageria](../bottlenecks.md#mensageria-kafka).

É um serviço **Spring Boot (Web MVC) + Spring Data JPA** sobre **PostgreSQL**, com o schema criado pelo
**Flyway**. Segue o Hands-on 1 do curso, com a rota de login do Hands-on 3.

## Dois repositórios: interface e implementação

Como no curso, o serviço tem **dois** repositórios:

| Repositório | Conteúdo | Quem usa |
|---|---|---|
| `account` | `AccountController` (interface com as rotas, anotada com `@FeignClient`) e os DTOs `AccountIn` e `AccountOut` | o `account-service` (implementa) e o `auth-service` (chama) |
| `account-service` | `AccountResource`, `AccountService`, JPA, Flyway, outbox, Dockerfile e `k8s/` | vira a imagem `account-service` |

A interface vira uma biblioteca Maven (`store:account:1.0.0`). O **contrato fica num lugar só**: o
`AccountResource` implementa o `AccountController`, e o Auth usa a mesma interface como cliente Feign. Se
uma rota mudar, os dois lados param de compilar até serem atualizados.

``` { .java .copy linenums="1" title="AccountController.java (pna26/account)" }
--8<-- "https://raw.githubusercontent.com/pna26/account/8ccda4d03b37738ffe8b4947cc463497b086cb1e/src/main/java/store/account/AccountController.java"
```

## Rotas

| Método | Rota | Entrada | Saída |
|---|---|---|---|
| POST | `/accounts` | `{ name, email, password }` | `201` + `Location: /accounts/{id}` |
| GET | `/accounts` | — | `[{ id, name, email, role }]` |
| GET | `/accounts/{id}` | — | `{ id, name, email, role }` ou `404` |
| DELETE | `/accounts/{id}` | — | `204` |
| POST | `/accounts/login` | `{ email, password }` | `{ id, name, email, role }` ou `401` (interno: quem chama é o Auth) |
| GET | `/accounts/health-check` | — | `200` |

Pelo gateway, todas as rotas de `/accounts/**` exigem o cookie do token e passam pelo RBAC. Para criar
conta sem estar logado, o caminho é o `POST /auth/register` do [Auth](auth.md).

## Regra de negócio

O `AccountService` valida a entrada, calcula o hash da senha, força o papel `USER` e grava a conta e o
evento **na mesma transação**:

``` { .java .copy linenums="1" title="AccountService.java (pna26/account-service)" }
--8<-- "https://raw.githubusercontent.com/pna26/account-service/a66e0b39c87865509c30f195f536b78365831186/src/main/java/store/account/AccountService.java"
```

O login faz o caminho inverso: calcula o hash da senha recebida e procura uma conta com aquele e-mail
**e** aquele hash. Se não achar, responde `401` sem dizer se o erro foi no e-mail ou na senha.

## Banco (Flyway)

As migrations criam o schema e evoluem a tabela sem mexer à mão no banco:

=== "Tabela de contas"

    ``` { .sql title="V2026.08.24.002__create_table.sql" }
    --8<-- "https://raw.githubusercontent.com/pna26/account-service/a66e0b39c87865509c30f195f536b78365831186/src/main/resources/db/migration/V2026.08.24.002__create_table.sql"
    ```

=== "Outbox (mensageria)"

    ``` { .sql title="V2026.11.08.001__create_outbox.sql" }
    --8<-- "https://raw.githubusercontent.com/pna26/account-service/a66e0b39c87865509c30f195f536b78365831186/src/main/resources/db/migration/V2026.11.08.001__create_outbox.sql"
    ```

=== "Papel (RBAC)"

    ``` { .sql title="V2026.11.16.001__add_role.sql" }
    --8<-- "https://raw.githubusercontent.com/pna26/account-service/a66e0b39c87865509c30f195f536b78365831186/src/main/resources/db/migration/V2026.11.16.001__add_role.sql"
    ```

No compose o banco é um `postgres:17`; no EKS é o RDS PostgreSQL `rds-store` (ver [EKS](eks.md)). Só o
`DATABASE_HOST` do ConfigMap muda.

## Como foi testado

- Todas as rotas no compose local (cadastro, listagem, busca, `404`, `DELETE`, login certo e errado).
- Pelo gateway: cadastro e login de ponta a ponta no `scripts/smoke-test.sh`, que roda no CI a cada push.
- No EKS: o Flyway criou o schema no RDS no primeiro deploy (08/11) e o CI fez o deploy sozinho em 16/11.

## Limitações conhecidas

| Limite | Melhoria |
|---|---|
| E-mail já cadastrado devolve `500` (comportamento do material do curso) | tratar a violação de unicidade e devolver `409 Conflict` |
| Hash com SHA-256 sem *salt* (como no hands-on) | `BCrypt` ou `Argon2`, que são lentos de propósito e têm *salt* |
