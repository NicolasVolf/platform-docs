# Auth

!!! info "Repositórios: [pna26/auth](https://github.com/pna26/auth){target='_blank'} (interface) e [pna26/auth-service](https://github.com/pna26/auth-service){target='_blank'} (implementação)"

    Página completa do grupo: [Auth](https://pna26.github.io/platform/servicos/auth/){target="_blank"}

O Auth cuida da **identidade** do usuário. Ele:

1. **cadastra** (`POST /auth/register`), repassando ao [Account](account.md);
2. faz o **login** (`POST /auth/login`): pede ao Account para conferir e-mail e senha e, se der certo,
   gera um **JWT** e o devolve no cookie `__store_jwt_token`;
3. **resolve** o token para o gateway (`POST /auth/solve`): confere a assinatura e a validade e devolve
   o id da conta e o papel;
4. devolve os dados de quem está logado (`GET /auth/me`) e faz o **logout** (apaga o cookie).

O Auth **não tem banco**: quem guarda as contas e os hashes é o Account. É um serviço **Spring Boot (Web
MVC) + OpenFeign**, com o JWT feito pela biblioteca **JJWT 0.13**. Segue o Hands-on 3 do curso.

## Rotas

| Método | Rota | Entrada | Saída | Token no gateway? |
|---|---|---|---|---|
| POST | `/auth/register` | `{ name, email, password }` | `200` | não |
| POST | `/auth/login` | `{ email, password }` | `200` + `Set-Cookie: __store_jwt_token=<jwt>` ou `401` | não |
| GET | `/auth/logout` | — | `200` + cookie vazio com `Max-Age=0` | não |
| GET | `/auth/me` | header `id-account` (posto pelo gateway) | `{ id, name, email }` | sim |
| POST | `/auth/solve` | `{ token }` | `{ idAccount, role }` ou `401` | só o gateway chama, por dentro |
| GET | `/auth/health-check` | — | `200` | sim |

## Login e uso do token

``` mermaid
sequenceDiagram
    actor U as Usuário
    participant G as gateway
    participant A as auth
    participant C as account
    U->>G: POST /auth/login {email, password}
    G->>A: (rota aberta)
    A->>C: POST /accounts/login (Feign)
    C-->>A: {id, name, email, role} ou 401
    A->>A: JWT assinado (HMAC-SHA256)
    A-->>U: 200 Set-Cookie: __store_jwt_token
    U->>G: GET /accounts/{id} (cookie)
    G->>A: POST /auth/solve {token}
    A->>A: confere assinatura, nbf e exp
    A-->>G: {idAccount, role}
    G->>C: GET /accounts/{id} + id-account
```

O gateway guarda o resultado do `/auth/solve` no Redis (bottleneck de cache do Leonardo), então o Auth
não é chamado em toda requisição.

## JWT

| Campo | Valor |
|---|---|
| `jti` (id) | id da conta: é o que o `/auth/solve` devolve |
| `sub` | nome da conta |
| `email` | e-mail da conta |
| `role` | `USER` ou `ADMIN` (ver [RBAC](../bottlenecks.md#autorizacao-rbac)) |
| `iss` | `Insper::PMA` |
| `nbf` / `exp` | agora / agora + `JWT_DURATION` |
| Assinatura | HMAC-SHA256 com a chave `JWT_SECRET_KEY` (base64 de 32 bytes) |

O token **não leva senha** e não é criptografado: qualquer um decodifica o conteúdo. O que impede a
falsificação é a **assinatura**. Trocar `USER` por `ADMIN` no payload invalida a assinatura e o
`/auth/solve` responde `401` (há um teste para isso no `JwtServiceTest`).

``` { .java .copy linenums="1" title="JwtService.java (pna26/auth-service)" }
--8<-- "https://raw.githubusercontent.com/pna26/auth-service/9edce6dc954f4528f67bf4555f305edab38e58af/src/main/java/store/auth/JwtService.java"
```

### Cookie

| Atributo | Por quê |
|---|---|
| `HttpOnly` | o JavaScript da página não lê o cookie, então um XSS não consegue roubar o token |
| `Secure` | o navegador só manda o cookie por HTTPS (ou para `localhost`) |
| `SameSite=None` | permite que um front-end em outra origem chame a API com o cookie (exige `Secure`) |
| `Path=/`, `Max-Age` = duração do token | o cookie vale para todas as rotas e some junto com o token |

## Código

=== "AuthController.java (interface)"

    ``` { .java .copy linenums="1" }
    --8<-- "https://raw.githubusercontent.com/pna26/auth/cf1ddb39a264d0cea2326f85e1cc0e07149bb630/src/main/java/store/auth/AuthController.java"
    ```

=== "AuthService.java"

    ``` { .java .copy linenums="1" }
    --8<-- "https://raw.githubusercontent.com/pna26/auth-service/9edce6dc954f4528f67bf4555f305edab38e58af/src/main/java/store/auth/AuthService.java"
    ```

=== "AuthResource.java"

    ``` { .java .copy linenums="1" }
    --8<-- "https://raw.githubusercontent.com/pna26/auth-service/9edce6dc954f4528f67bf4555f305edab38e58af/src/main/java/store/auth/AuthResource.java"
    ```

## Decisões

| Decisão | Motivo |
|---|---|
| Auth sem banco, conferindo a senha no Account | Um só dono dos dados das contas. O Auth nunca vê o hash. |
| JWT *stateless* | Para validar só precisa da chave, sem banco. Qualquer réplica do Auth valida qualquer token. |
| Login errado → `401` | O `AuthService` converte o `401`/`400` do Account (que chega como `FeignException`) em `401 Invalid credentials`. |
| `JWT_SECRET_KEY` **obrigatória** | Não existe chave padrão no código: sem a variável, o Auth não sobe. Uma chave num repositório público deixaria qualquer pessoa gerar tokens válidos. No compose ela vem do `.env`; no EKS, do Secret `auth-secrets`. |
| JJWT com versão fixa (0.13.0) | A API do JJWT mudou bastante entre versões; com a versão fixa o build não quebra sozinho. |

## Como foi testado

- No compose: register, login (cookie `HttpOnly`/`Secure`/`SameSite=None` e claims corretos), solve,
  token adulterado → `401`, `me` e logout.
- No CI: `JwtServiceTest` (4 testes, incluindo a troca de papel no payload) e o smoke test de ponta a ponta.
- No EKS (16/11/2026, deploy pelo CI): register `200`, login `200` com o cookie, senha errada `401`.

## Limitações conhecidas

| Limite | Como ficaria em produção |
|---|---|
| Erro do Account no cadastro (e-mail duplicado, senha curta) chega como `500` | tratar a `FeignException` e repassar o status do Account (`400`, `409`) |
| Logout só apaga o cookie: o token continua válido até o `exp` | lista de tokens revogados (por `jti`) no Redis, consultada pelo gateway |
| O EKS atende só por HTTP, e o navegador não guarda um cookie `Secure` sem HTTPS | domínio próprio + certificado (ACM) no load balancer; documentado como limitação em [Arquitetura](https://pna26.github.io/platform/arquitetura/#https-limitacao-conhecida){target="_blank"} |
| O Auth foi *OOMKilled* no ensaio do teste com HPA | `-XX:MaxRAMPercentage=60` ou limite de memória de 768 Mi |
