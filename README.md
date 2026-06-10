<h1 align="center">
  <a name="logo" href="https://www.gira-bicicletasdelisboa.pt/">
    <img src="https://www.gira-bicicletasdelisboa.pt/wp-content/themes/gira/resources/assets/images/logo_gira.svg" alt="Logótipo GIRA" width="300">
  </a>
  <br>
  GIRA API Documentation
</h1>

<h4 align="center">Documentação da API privada da GIRA, usada pela aplicação.</h4>
<h6 align="center">
  Esta documentação descreve a versão actualmente observada na aplicação GIRA/GIRA+.
  <br>
  A autenticação é feita através da EMEL. Os access tokens expiram rapidamente e devem ser renovados com refresh.
</h6>

<hr>

## Índice de Conteúdos

<details>
<summary>Clique para expandir</summary>

- [Estado Actual](#estado-actual)
- [Autenticação](#auth)
  * [Login](#auth-login)
  * [Refresh](#auth-refresh)
  * [Revoke](#auth-revoke)
  * [Perfil](#auth-profile)
- [API GIRA GraphQL](#gira-api)
  * [Pedido HTTP](#gira-api-http)
  * [Queries Operacionais](#gira-api-queries-operacionais)
  * [Queries de Conta](#gira-api-queries-conta)
  * [Mutations](#gira-api-mutations)
- [WebSocket](#gira-ws)
  * [Subscriptions Públicas](#gira-ws-publicas)
  * [Subscriptions com Access Token](#gira-ws-token)
- [Schema](#schema)
- [Notas de Segurança](#seguranca)

</details>

<hr>

<h1 id="estado-actual">Estado Actual</h1>

A app analisada usa estes endpoints para a GIRA:

| Área | URL |
| --- | --- |
| Login | `https://c2g091p01.emel.pt/auth/login` |
| Refresh | `https://c2g091p01.emel.pt/auth/token/refresh` |
| Revoke | `https://c2g091p01.emel.pt/auth/token/revoke` |
| Perfil | `https://c2g091p01.emel.pt/auth/user` |
| GraphQL HTTP | `https://c2g091p01.emel.pt/ws/graphql` |
| GraphQL WebSocket | `wss://c2g091p01.emel.pt/ws/graphql` |

O endpoint antigo `https://apigira.emel.pt/graphql` não é mais utilizado. O endpoint ws agora usado é:

```http
https://c2g091p01.emel.pt/ws/graphql
```

<hr>

<h1 id="auth">Autenticação</h1>

<h2 id="auth-login">Login</h2>

Autenticação através do sistema de login da EMEL.

### Pedido

`POST https://c2g091p01.emel.pt/auth/login`

Headers observados:

```http
User-Agent: Gira/3.4.3 (Android 34)
Content-Type: application/json
Priority: high
```

Body:

```json
{
  "Provider": "EmailPassword",
  "CredentialsEmailPassword": {
    "email": "email@example.com",
    "password": "password"
  }
}
```

### Resposta

```json
{
  "error": {
    "code": 0,
    "message": "Success"
  },
  "data": {
    "accessToken": "jwt",
    "refreshToken": "refresh-token",
    "expiration": 639166398323656800
  }
}
```

<h3>JWT</h3>

O `accessToken` é um JWT. Pode ser descodificado em [jwt.io](https://jwt.io/).

<h3>Ticks .NET</h3>

O campo `expiration` vem em ticks .NET.

- 1 tick = 100 nanossegundos.
- A contagem começa em `0001-01-01T00:00:00.0000000 UTC`.
- Conversão para Unix milliseconds:

```js
const unixMs = Number((BigInt(ticks) - 621355968000000000n) / 10000n);
```

<h2 id="auth-refresh">Refresh</h2>

`POST https://c2g091p01.emel.pt/auth/token/refresh`

Body:

```json
{
  "token": "refresh-token"
}
```

<h2 id="auth-revoke">Revoke</h2>

`POST https://c2g091p01.emel.pt/auth/token/revoke`

Body:

```json
{
  "token": "refresh-token-ou-access-token"
}
```

<h2 id="auth-profile">Perfil</h2>

`GET https://c2g091p01.emel.pt/auth/user`

Headers:

```http
Authorization: Bearer {accessToken}
User-Agent: Gira/3.4.3 (Android 34)
Content-Type: application/json
```

<hr>

<h1 id="gira-api">API GIRA GraphQL</h1>

A API GIRA usa GraphQL. O endpoint HTTP validado que não precisa do token da firebase é:

```http
POST https://c2g091p01.emel.pt/ws/graphql
```

<h2 id="gira-api-http">Pedido HTTP</h2>

Headers:

```http
Authorization: Bearer {accessToken}
User-Agent: Gira/3.4.3 (Android 34)
Content-Type: application/json
```

Body:

```json
{
  "query": "{ __typename }"
}
```

Resposta validada:

```json
{
  "data": {
    "__typename": "Query"
  }
}
```

Sem `Authorization: Bearer`, o endpoint HTTP devolve `401 Unauthorized`.

<h2 id="gira-api-queries-operacionais">Queries Operacionais</h2>

Estas queries funcionam com Bearer EMEL e sem o token da firebase.

| Query | Descrição |
| --- | --- |
| `serviceStatus` | Estado global do serviço. |
| `getServerTime` | Data/hora do servidor. |
| `getStations` | Lista estações, coordenadas, estado e disponibilidade. |
| `getSubscriptions` | Lista passes/subscrições disponíveis. |
| `getDocks(input: String)` | Lista docas de uma estação. O input é o `serialNumber` da estação. |
| `getBikes(input: String)` | Lista bicicletas de uma estação. O input é o `serialNumber` da estação. |

Exemplo:

```graphql
query {
  getStations {
    code
    serialNumber
    name
    latitude
    longitude
    bikes
    docks
    assetStatus
    stype
  }
}
```

<h2 id="gira-api-queries-conta">Queries de Conta</h2>

Estas queries também funcionam sem Firebase, mas devolvem dados da conta autenticada. Devem ser tratadas como sensíveis.

| Query | Descrição |
| --- | --- |
| `activeTrip` | Viagem activa do utilizador. |
| `activeTripCost` | Custo da viagem activa. Se não houver viagem, pode devolver `trip_not_found`. |
| `getTrip(input: String)` | Detalhe de viagem por código. |
| `tripHistory(pageInput: PageInput)` | Histórico de viagens. |
| `unratedTrips(pageInput: PageInput)` | Viagens por avaliar. |
| `canPayTripWithPoints(input: String)` | Verifica se uma viagem pode ser paga com pontos. |
| `activeUserSubscriptions` | Subscrição activa no formato usado pela app. |
| `activeSubscriptions` | Subscrições activas. |
| `inactiveSubscriptions` | Subscrições inactivas. |
| `paidSubscriptions` | Subscrições pagas. |
| `newPaidSubscriptions` | Novas subscrições pagas. |
| `newPendingPaymentSubscriptions` | Subscrições pendentes de pagamento. |
| `subscriptionHistory(pageInput: PageInput)` | Histórico de subscrições. |
| `client` | Dados de cliente, saldo e pontos. |
| `invoice` | Faturas. |
| `transaction` | Transacções. |
| `clientCreditCard` | Cartões associados. |
| `getCreditCards(input: String)` | Cartões guardados. |
| `promotionInstance` | Promoções aplicáveis. |

<h2 id="gira-api-mutations">Mutations</h2>

As mutations existem no schema, mas não devem ser executadas sem confirmação explícita, porque podem reservar bicicletas, iniciar viagens, alterar pagamentos ou modificar a conta.

| Mutation | Descrição |
| --- | --- |
| `reserveBike(input: String)` | Reserva bicicleta. |
| `cancelBikeReserve` | Cancela reserva activa. |
| `startTrip` | Inicia viagem. |
| `rateTrip(in: RateTrip_In)` | Avalia viagem. |
| `tripPayWithPoints(input: String)` | Paga viagem com pontos. |
| `tripPayWithNoPoints(input: String)` | Paga viagem sem pontos. |
| `registerNavigatorCard(input: String)` | Associa cartão Navegante. |
| `clearNavigatorCard(input: String)` | Remove cartão Navegante. |
| `createUserSubscription(input: String)` | Cria subscrição. |
| `subscriptionEasyPay(in: SubscriptionEasyPay_In)` | Pagamento/subscrição via cartão. |
| `subscriptionTopUpPayPalBond(in: SubscriptionTopUpPayPalBond_In)` | Pagamento/caução via PayPal. |
| `subscriptionTopUpPayPal(input: String)` | Top-up PayPal. |
| `subscriptionWallet(input: String)` | Pagamento via carteira/saldo. |
| `acceptTermsAndConditions` | Aceita termos e condições. |
| `createCreditCardFrequentPayment` | Cria pagamento recorrente com cartão. |
| `updateCreditCardInfo(input: String)` | Actualiza cartão. |
| `removeCreditCard(input: String)` | Remove cartão. |
| `creditCardTopUp(in: CreditCardTopUp_In)` | Carregamento por cartão. |
| `insertPromotionalCode(input: String)` | Aplica código promocional. |
| `deleteClient` | Apaga cliente. |
| `registerClient(input: String)` | Regista cliente. |
| `validateLogin(in: ValidateLogin_In)` | Valida login/termos/RGPD. |
| `topUpPayPal(input: Float)` | Carregamento PayPal. |
| `updatePayPalAccount(input: String)` | Actualiza conta PayPal. |
| `clearPayPalAccount` | Remove conta PayPal. |
| `emailInvoices(in: EmailInvoices_In)` | Envia faturas por email. |
| `emailReturns(in: EmailReturns_In)` | Envia devoluções por email. |

<hr>

<h1 id="gira-ws">WebSocket</h1>

Endpoint:

```http
wss://c2g091p01.emel.pt/ws/graphql
Sec-WebSocket-Protocol: graphql-ws
```

Handshake mínimo validado:

```json
{
  "type": "connection_init",
  "payload": {}
}
```

Resposta:

```json
{
  "id": null,
  "type": "connection_ack",
  "payload": null
}
```

<h2 id="gira-ws-publicas">Subscriptions Públicas</h2>

Estas subscriptions funcionaram sem Bearer e sem Firebase/device token.

```graphql
subscription {
  serverDate {
    date
  }
}
```

```graphql
subscription {
  operationalStationsSubscription {
    code
    name
    bikes
    docks
    assetStatus
    stype
  }
}
```

Resultado observado:

- `serverDate`: OK.
- `operationalStationsSubscription`: OK, devolveu 203 estações.

<h2 id="gira-ws-token">Subscriptions com Access Token</h2>

`activeTripSubscription` precisa de `_access_token` nas variáveis. Sem token devolve um erro: `error: 401`.

Exemplo:

```json
{
  "type": "start",
  "id": "active-trip",
  "payload": {
    "operationName": "activeTripSubscription",
    "query": "subscription activeTripSubscription($_access_token: String) { activeTripSubscription(_access_token: $_access_token) { code bike startDate endDate cost finished canPayWithMoney canUsePoints clientPoints tripPoints canceled period periodTime error } }",
    "variables": {
      "_access_token": "{accessToken}"
    }
  }
}
```

Resposta observada sem viagem activa:

```json
{
  "data": {
    "activeTripSubscription": {
      "code": "no_trip",
      "bike": "dummy",
      "finished": true,
      "canceled": true,
      "error": 0
    }
  }
}
```
